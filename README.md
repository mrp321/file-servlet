[TOC]

# FileServlet supporting resume and caching and GZIP

## NOTICE - NEWER VERSION AVAILABLE!
Since OmniFaces 2.2, the below file servlet has been reworked, modernized and refactored into a highly reusable abstract org.omnifaces.servlet.FileServlet class in JSF utility library OmniFaces.


## Introduction
In the almost 2 year old FileServlet and ImageServlet articles you can find basic examples of a download servlet and an image servlet. It does in fact nothing more than obtaining an InputStream of the desired resource/file and writing it to the OutputStream of the HTTP response along with a set of important response headers. It does not support resumes and effective caching of client side data.

If one downloaded a big file and got network problems on 99% of the file, one wouldn't be happy to discover the need to download the complete file again after getting network back. If a browser decides to check the cached images for changes, it would send a HEAD request to determine under each the unique file identifier and its timestamp or it would send a conditional GET request to determine the response status. If the image isn't changed according to the server response, the client won't re-request the image again to save the network bandwidth and other efforts.

You could leverage the task to a default servlet of the webcontainer/appserver you're using, but most of them doesn't implement all of the performance enhancements, so does for example Tomcat's DefaultServlet not support the Expires header.


## Resume downloads
To enable download resumes, the server have to send at least the Accept-Ranges, ETag and Last-Modified response headers to the client along with the file.

The Accept-Ranges response header with the value "bytes" informs the client that the server supports byte-range requests. With this the client could request for a specific byte range using the Range request header.

The ETag response header should contain a value which represents an unique identifier of the file in question so that both the server and the client can identify the file. You can use a combination of the file name, file size and file modification timestap for this. Some servers hauls this combination through a MD5 function to get an unique 32 character hexadecimal string. But this is not necessarily unique because two different strings could generate the same MD5 hash, so we won't use it here. The client could resend the obtained ETag back to the server for validation using the If-Match or If-Range request headers.

The Last-Modified response header should contain a date which represents the last modification timestamp of the file as it is at the server side. The client could resend the obtained timestamp back to the server for validation using the If-Unmodified-Since or If-Range request headers. Important note: keep in mind that the timestamp accuracy in server side Java is in milliseconds while the accurancy of the Last-Modified header is in seconds. In Java code you should add 1 second (1000ms) to the value of the If-* request headers to bridge this difference before validation.

Whenever the client sends a partial GET request with a Range request header to the server, then server should intercept on the conditional GET request headers (all headers starting with If) and handle accordingly. Whenever the If-Match or If-Unmodified-Since conditions are negative, the server should send a 412 "Precondition Failed" response back without any content. Whenever the If-Range condition is negative, then the server should ignore the Range header and send the full file back. Whenever the Range header is in invalid format, then the server should send a 416 "Requested Range Not Satisfiable" response back without any content.

If a partial GET request with a valid Range header is sent by the client, then the server should send the specific byte range(s) back as a 206 "Partial Content" response.


## Client side caching
The principle is the same as with resume downloads, with the only difference that no Range request header is been sent to the server. The server only have to check and validate any conditional GET request headers and respond accordingly. Usually those are the If-None-Match or If-Modified-Since request headers. The client could also send a HEAD request (for which the server should respond exactly like a GET, but completely without content) and determine the obtained ETag and Last-Modified response headers itself.

Whenever the If-None-Match or If-Modified-Since conditions are positive, the server should send a 304 "Not Modified" response back without any content. If this happens, then the client is allowed to use the content which is already available in the client side cache.

Further on you can use the Expires response header to inform the client how long to keep the content in the client side cache without firing any request about that, even no HEAD requests.

## GZIP compression
To save more network bandwitch, we could compress text files (text/javascript, text/css, text/xml, text/csv, etcetera) with GZIP. Generally you can save up to 70% of network bandwidth by compressing text files with GZIP. We only need to check if the client accepts GZIP encoding by checking if the Accept-Encoding header contains "gzip". If this is true, and the client is requesting the full file, then the full text file will be compressed. Statistics learn that about 90% of the browsers supports GZIP.

This may also be possible for all files other than text, but as it usually concerns images and another kinds of (large) binary files, it may unnecessarily generate too much overhead to (de)compress them.

## The Code
OK, enough boring technical background blah, now on to the code!

This fileservlet does everything what it should do based on the request headers as described above. It also supports multipart byte requests (the client could send multiple ranges commaseparated along with the Range header). The whole stuff is targeted on at least Java EE 5 and developed and tested in Eclipse 3.4 with Tomcat 6. It is tested with different webbrowsers (FireFox2/3, IE6/7/8, Opera8/9, Safari2/3 and Chrome) and also with a plain vanilla Java Application using URLConnection.

You can use it for any file types: binary files, text files, images, etcetera. When the requested file is a text file or an image or when its content type is covered by the Accept request header of the client, then it will be displayed inline, otherwise it will be sent as an attachment which will pop up a 'save as' dialogue.

It's almost 485 lines of code of which the nearly half are less or more rudimentary due to comments (read them all though), long-code line breaks and blank lines. You can just copy'n'paste and run it. You're free to make changes whenever needed as long as it's not for commercial use.


```java
/*
 * net/balusc/webapp/FileServlet.java
 *
 * Copyright (C) 2009 BalusC
 *
 * This program is free software: you can redistribute it and/or modify it under the terms of the
 * GNU Lesser General Public License as published by the Free Software Foundation, either version 3
 * of the License, or (at your option) any later version.
 * 
 * This library is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without
 * even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
 * Lesser General Public License for more details.
 * 
 * You should have received a copy of the GNU Lesser General Public License along with this library.
 * If not, see <http://www.gnu.org/licenses/>.
 */

package net.balusc.webapp;

import java.io.Closeable;
import java.io.File;
import java.io.IOException;
import java.io.OutputStream;
import java.io.RandomAccessFile;
import java.net.URLDecoder;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.zip.GZIPOutputStream;

import javax.servlet.ServletException;
import javax.servlet.ServletOutputStream;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * A file servlet supporting resume of downloads and client-side caching and GZIP of text content.
 * This servlet can also be used for images, client-side caching would become more efficient.
 * This servlet can also be used for text files, GZIP would decrease network bandwidth.
 *
 * @author BalusC
 * @link http://balusc.blogspot.com/2009/02/fileservlet-supporting-resume-and.html
 */
public class FileServlet extends HttpServlet {

    // Constants ----------------------------------------------------------------------------------

    private static final int DEFAULT_BUFFER_SIZE = 10240; // ..bytes = 10KB.
    private static final long DEFAULT_EXPIRE_TIME = 604800000L; // ..ms = 1 week.
    private static final String MULTIPART_BOUNDARY = "MULTIPART_BYTERANGES";

    // Properties ---------------------------------------------------------------------------------

    private String basePath;

    // Actions ------------------------------------------------------------------------------------

    /**
     * Initialize the servlet.
     * @see HttpServlet#init().
     */
    public void init() throws ServletException {

        // Get base path (path to get all resources from) as init parameter.
        this.basePath = getInitParameter("basePath");

        // Validate base path.
        if (this.basePath == null) {
            throw new ServletException("FileServlet init param 'basePath' is required.");
        } else {
            File path = new File(this.basePath);
            if (!path.exists()) {
                throw new ServletException("FileServlet init param 'basePath' value '"
                    + this.basePath + "' does actually not exist in file system.");
            } else if (!path.isDirectory()) {
                throw new ServletException("FileServlet init param 'basePath' value '"
                    + this.basePath + "' is actually not a directory in file system.");
            } else if (!path.canRead()) {
                throw new ServletException("FileServlet init param 'basePath' value '"
                    + this.basePath + "' is actually not readable in file system.");
            }
        }
    }

    /**
     * Process HEAD request. This returns the same headers as GET request, but without content.
     * @see HttpServlet#doHead(HttpServletRequest, HttpServletResponse).
     */
    protected void doHead(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException
    {
        // Process request without content.
        processRequest(request, response, false);
    }

    /**
     * Process GET request.
     * @see HttpServlet#doGet(HttpServletRequest, HttpServletResponse).
     */
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException
    {
        // Process request with content.
        processRequest(request, response, true);
    }

    /**
     * Process the actual request.
     * @param request The request to be processed.
     * @param response The response to be created.
     * @param content Whether the request body should be written (GET) or not (HEAD).
     * @throws IOException If something fails at I/O level.
     */
    private void processRequest
        (HttpServletRequest request, HttpServletResponse response, boolean content)
            throws IOException
    {
        // Validate the requested file ------------------------------------------------------------

        // Get requested file by path info.
        String requestedFile = request.getPathInfo();

        // Check if file is actually supplied to the request URL.
        if (requestedFile == null) {
            // Do your thing if the file is not supplied to the request URL.
            // Throw an exception, or send 404, or show default/warning page, or just ignore it.
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        // URL-decode the file name (might contain spaces and on) and prepare file object.
        File file = new File(basePath, URLDecoder.decode(requestedFile, "UTF-8"));

        // Check if file actually exists in filesystem.
        if (!file.exists()) {
            // Do your thing if the file appears to be non-existing.
            // Throw an exception, or send 404, or show default/warning page, or just ignore it.
            response.sendError(HttpServletResponse.SC_NOT_FOUND);
            return;
        }

        // Prepare some variables. The ETag is an unique identifier of the file.
        String fileName = file.getName();
        long length = file.length();
        long lastModified = file.lastModified();
        String eTag = fileName + "_" + length + "_" + lastModified;
        long expires = System.currentTimeMillis() + DEFAULT_EXPIRE_TIME;


        // Validate request headers for caching ---------------------------------------------------

        // If-None-Match header should contain "*" or ETag. If so, then return 304.
        String ifNoneMatch = request.getHeader("If-None-Match");
        if (ifNoneMatch != null && matches(ifNoneMatch, eTag)) {
            response.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
            response.setHeader("ETag", eTag); // Required in 304.
            response.setDateHeader("Expires", expires); // Postpone cache with 1 week.
            return;
        }

        // If-Modified-Since header should be greater than LastModified. If so, then return 304.
        // This header is ignored if any If-None-Match header is specified.
        long ifModifiedSince = request.getDateHeader("If-Modified-Since");
        if (ifNoneMatch == null && ifModifiedSince != -1 && ifModifiedSince + 1000 > lastModified) {
            response.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
            response.setHeader("ETag", eTag); // Required in 304.
            response.setDateHeader("Expires", expires); // Postpone cache with 1 week.
            return;
        }


        // Validate request headers for resume ----------------------------------------------------

        // If-Match header should contain "*" or ETag. If not, then return 412.
        String ifMatch = request.getHeader("If-Match");
        if (ifMatch != null && !matches(ifMatch, eTag)) {
            response.sendError(HttpServletResponse.SC_PRECONDITION_FAILED);
            return;
        }

        // If-Unmodified-Since header should be greater than LastModified. If not, then return 412.
        long ifUnmodifiedSince = request.getDateHeader("If-Unmodified-Since");
        if (ifUnmodifiedSince != -1 && ifUnmodifiedSince + 1000 <= lastModified) {
            response.sendError(HttpServletResponse.SC_PRECONDITION_FAILED);
            return;
        }


        // Validate and process range -------------------------------------------------------------

        // Prepare some variables. The full Range represents the complete file.
        Range full = new Range(0, length - 1, length);
        List<Range> ranges = new ArrayList<Range>();

        // Validate and process Range and If-Range headers.
        String range = request.getHeader("Range");
        if (range != null) {

            // Range header should match format "bytes=n-n,n-n,n-n...". If not, then return 416.
            if (!range.matches("^bytes=\\d*-\\d*(,\\d*-\\d*)*$")) {
                response.setHeader("Content-Range", "bytes */" + length); // Required in 416.
                response.sendError(HttpServletResponse.SC_REQUESTED_RANGE_NOT_SATISFIABLE);
                return;
            }

            // If-Range header should either match ETag or be greater then LastModified. If not,
            // then return full file.
            String ifRange = request.getHeader("If-Range");
            if (ifRange != null && !ifRange.equals(eTag)) {
                try {
                    long ifRangeTime = request.getDateHeader("If-Range"); // Throws IAE if invalid.
                    if (ifRangeTime != -1 && ifRangeTime + 1000 < lastModified) {
                        ranges.add(full);
                    }
                } catch (IllegalArgumentException ignore) {
                    ranges.add(full);
                }
            }

            // If any valid If-Range header, then process each part of byte range.
            if (ranges.isEmpty()) {
                for (String part : range.substring(6).split(",")) {
                    // Assuming a file with length of 100, the following examples returns bytes at:
                    // 50-80 (50 to 80), 40- (40 to length=100), -20 (length-20=80 to length=100).
                    long start = sublong(part, 0, part.indexOf("-"));
                    long end = sublong(part, part.indexOf("-") + 1, part.length());

                    if (start == -1) {
                        start = length - end;
                        end = length - 1;
                    } else if (end == -1 || end > length - 1) {
                        end = length - 1;
                    }

                    // Check if Range is syntactically valid. If not, then return 416.
                    if (start > end) {
                        response.setHeader("Content-Range", "bytes */" + length); // Required in 416.
                        response.sendError(HttpServletResponse.SC_REQUESTED_RANGE_NOT_SATISFIABLE);
                        return;
                    }

                    // Add range.
                    ranges.add(new Range(start, end, length));
                }
            }
        }


        // Prepare and initialize response --------------------------------------------------------

        // Get content type by file name and set default GZIP support and content disposition.
        String contentType = getServletContext().getMimeType(fileName);
        boolean acceptsGzip = false;
        String disposition = "inline";

        // If content type is unknown, then set the default value.
        // For all content types, see: http://www.w3schools.com/media/media_mimeref.asp
        // To add new content types, add new mime-mapping entry in web.xml.
        if (contentType == null) {
            contentType = "application/octet-stream";
        }

        // If content type is text, then determine whether GZIP content encoding is supported by
        // the browser and expand content type with the one and right character encoding.
        if (contentType.startsWith("text")) {
            String acceptEncoding = request.getHeader("Accept-Encoding");
            acceptsGzip = acceptEncoding != null && accepts(acceptEncoding, "gzip");
            contentType += ";charset=UTF-8";
        } 

        // Else, expect for images, determine content disposition. If content type is supported by
        // the browser, then set to inline, else attachment which will pop a 'save as' dialogue.
        else if (!contentType.startsWith("image")) {
            String accept = request.getHeader("Accept");
            disposition = accept != null && accepts(accept, contentType) ? "inline" : "attachment";
        }

        // Initialize response.
        response.reset();
        response.setBufferSize(DEFAULT_BUFFER_SIZE);
        response.setHeader("Content-Disposition", disposition + ";filename=\"" + fileName + "\"");
        response.setHeader("Accept-Ranges", "bytes");
        response.setHeader("ETag", eTag);
        response.setDateHeader("Last-Modified", lastModified);
        response.setDateHeader("Expires", expires);


        // Send requested file (part(s)) to client ------------------------------------------------

        // Prepare streams.
        RandomAccessFile input = null;
        OutputStream output = null;

        try {
            // Open streams.
            input = new RandomAccessFile(file, "r");
            output = response.getOutputStream();

            if (ranges.isEmpty() || ranges.get(0) == full) {

                // Return full file.
                Range r = full;
                response.setContentType(contentType);

                if (content) {
                    if (acceptsGzip) {
                        // The browser accepts GZIP, so GZIP the content.
                        response.setHeader("Content-Encoding", "gzip");
                        output = new GZIPOutputStream(output, DEFAULT_BUFFER_SIZE);
                    } else {
                        // Content length is not directly predictable in case of GZIP.
                        // So only add it if there is no means of GZIP, else browser will hang.
                        response.setHeader("Content-Length", String.valueOf(r.length));
                    }

                    // Copy full range.
                    copy(input, output, r.start, r.length);
                }

            } else if (ranges.size() == 1) {

                // Return single part of file.
                Range r = ranges.get(0);
                response.setContentType(contentType);
                response.setHeader("Content-Range", "bytes " + r.start + "-" + r.end + "/" + r.total);
                response.setHeader("Content-Length", String.valueOf(r.length));
                response.setStatus(HttpServletResponse.SC_PARTIAL_CONTENT); // 206.

                if (content) {
                    // Copy single part range.
                    copy(input, output, r.start, r.length);
                }

            } else {

                // Return multiple parts of file.
                response.setContentType("multipart/byteranges; boundary=" + MULTIPART_BOUNDARY);
                response.setStatus(HttpServletResponse.SC_PARTIAL_CONTENT); // 206.

                if (content) {
                    // Cast back to ServletOutputStream to get the easy println methods.
                    ServletOutputStream sos = (ServletOutputStream) output;

                    // Copy multi part range.
                    for (Range r : ranges) {
                        // Add multipart boundary and header fields for every range.
                        sos.println();
                        sos.println("--" + MULTIPART_BOUNDARY);
                        sos.println("Content-Type: " + contentType);
                        sos.println("Content-Range: bytes " + r.start + "-" + r.end + "/" + r.total);

                        // Copy single part range of multi part range.
                        copy(input, output, r.start, r.length);
                    }

                    // End with multipart boundary.
                    sos.println();
                    sos.println("--" + MULTIPART_BOUNDARY + "--");
                }
            }
        } finally {
            // Gently close streams.
            close(output);
            close(input);
        }
    }

    // Helpers (can be refactored to public utility class) ----------------------------------------

    /**
     * Returns true if the given accept header accepts the given value.
     * @param acceptHeader The accept header.
     * @param toAccept The value to be accepted.
     * @return True if the given accept header accepts the given value.
     */
    private static boolean accepts(String acceptHeader, String toAccept) {
        String[] acceptValues = acceptHeader.split("\\s*(,|;)\\s*");
        Arrays.sort(acceptValues);
        return Arrays.binarySearch(acceptValues, toAccept) > -1
            || Arrays.binarySearch(acceptValues, toAccept.replaceAll("/.*$", "/*")) > -1
            || Arrays.binarySearch(acceptValues, "*/*") > -1;
    }

    /**
     * Returns true if the given match header matches the given value.
     * @param matchHeader The match header.
     * @param toMatch The value to be matched.
     * @return True if the given match header matches the given value.
     */
    private static boolean matches(String matchHeader, String toMatch) {
        String[] matchValues = matchHeader.split("\\s*,\\s*");
        Arrays.sort(matchValues);
        return Arrays.binarySearch(matchValues, toMatch) > -1
            || Arrays.binarySearch(matchValues, "*") > -1;
    }

    /**
     * Returns a substring of the given string value from the given begin index to the given end
     * index as a long. If the substring is empty, then -1 will be returned
     * @param value The string value to return a substring as long for.
     * @param beginIndex The begin index of the substring to be returned as long.
     * @param endIndex The end index of the substring to be returned as long.
     * @return A substring of the given string value as long or -1 if substring is empty.
     */
    private static long sublong(String value, int beginIndex, int endIndex) {
        String substring = value.substring(beginIndex, endIndex);
        return (substring.length() > 0) ? Long.parseLong(substring) : -1;
    }

    /**
     * Copy the given byte range of the given input to the given output.
     * @param input The input to copy the given range to the given output for.
     * @param output The output to copy the given range from the given input for.
     * @param start Start of the byte range.
     * @param length Length of the byte range.
     * @throws IOException If something fails at I/O level.
     */
    private static void copy(RandomAccessFile input, OutputStream output, long start, long length)
        throws IOException
    {
        byte[] buffer = new byte[DEFAULT_BUFFER_SIZE];
        int read;

        if (input.length() == length) {
            // Write full range.
            while ((read = input.read(buffer)) > 0) {
                output.write(buffer, 0, read);
            }
        } else {
            // Write partial range.
            input.seek(start);
            long toRead = length;

            while ((read = input.read(buffer)) > 0) {
                if ((toRead -= read) > 0) {
                    output.write(buffer, 0, read);
                } else {
                    output.write(buffer, 0, (int) toRead + read);
                    break;
                }
            }
        }
    }

    /**
     * Close the given resource.
     * @param resource The resource to be closed.
     */
    private static void close(Closeable resource) {
        if (resource != null) {
            try {
                resource.close();
            } catch (IOException ignore) {
                // Ignore IOException. If you want to handle this anyway, it might be useful to know
                // that this will generally only be thrown when the client aborted the request.
            }
        }
    }

    // Inner classes ------------------------------------------------------------------------------

    /**
     * This class represents a byte range.
     */
    protected class Range {
        long start;
        long end;
        long length;
        long total;

        /**
         * Construct a byte range.
         * @param start Start of the byte range.
         * @param end End of the byte range.
         * @param total Total length of the byte source.
         */
        public Range(long start, long end, long total) {
            this.start = start;
            this.end = end;
            this.length = end - start + 1;
            this.total = total;
        }

    }

}
```


In order to get the FileServlet to work, add the following entries to the Web Deployment Descriptor web.xml:


```xml
<servlet>
    <servlet-name>fileServlet</servlet-name>
    <servlet-class>net.balusc.webapp.FileServlet</servlet-class>
    <init-param>
        <param-name>basePath</param-name>
        <param-value>/path/to/files</param-value>
    </init-param>
</servlet>

<servlet-mapping>
    <servlet-name>fileServlet</servlet-name>
    <url-pattern>/files/*</url-pattern>
</servlet-mapping>
The basePath value must represent the absolute path to a folder containing all those files. You can of course change the value of the basePath parameter and the url-pattern of the servlet-mapping to your taste.
```


Here are some basic use examples:

```html
<!-- XHTML or JSP -->
<a href="files/foo.exe">download foo.exe</a>
<a href="files/bar.zip">download bar.zip</a>

<img src="files/pic.jpg" />
<img src="files/logo.gif" />

<!-- JSF -->
<h:outputLink value="files/foo.exe">download foo.exe</h:outputLink>
<h:outputLink value="files/bar.zip">download bar.zip</h:outputLink>
<h:outputLink value="files/#{myBean.fileName}">
    <h:outputText value="download #{myBean.fileName}" />
</h:outputLink>

<h:graphicImage value="files/pic.jpg" />
<h:graphicImage value="files/logo.gif" />
<h:graphicImage value="files/#{myBean.imageFileName}" />
```

Important note: this servlet example does not take the requested file as request parameter, but just as part of the absolute URL, because a certain widely used browser developed by a team in Redmond would take the last part of the servlet URL path as filename during the 'Save As' dialogue instead of the in the headers supplied filename. Using the filename as part of the absolute URL (and thus not as request parameter) will fix this utterly stupid behaviour. As a bonus, the URL's look much nicer without query parameters.



**see more at : [FileServlet supporting resume and caching and GZIP](http://balusc.omnifaces.org/2009/02/fileservlet-supporting-resume-and.html)**
