# spring-request-response-body-auditing
Spring java based interceptor to audit request and response body

## Steps: ##
1. A filter to wrap the servlet request and response
2. Extending HttpServletRequestWrapper and HttpServletResponseWrapper
3. Extending ServletInputStream and ServletOutputStream
4. Creating an interceptor - HandlerInterceptor
5. Deifing the filter in web.xml

### A filter ###
```
@Component
public class RequestResponseWrapperFilter implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        // DO NOTHING
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        // Wrap the request and response
        ResponseWrapper responseWrapper = new ResponseWrapper((HttpServletResponse) response);
        RequestWrapper requestWrapper = new RequestWrapper((HttpServletRequest) request);
        chain.doFilter(requestWrapper, responseWrapper);
    }

    @Override
    public void destroy() {
        // DO NOTHING
    }
}
```
### Wrappers ###
#### Request Wrapper ####
You need `commons-io`. There are other ways to convert an `InputStream` to `byte[]`. But I feel this is easy :)
```
public class RequestWrapper extends HttpServletRequestWrapper{

    protected byte[] bytes;

    public RequestWrapper(HttpServletRequest request) throws IOException {
        super(request);
        bytes = IOUtils.toByteArray(request.getInputStream());
    }

    @Override
    public String toString() {
        try {
            return IOUtils.toString(bytes, CharEncoding.UTF_8);
        } catch (IOException e) {
            return StringUtils.EMPTY;
        }
    }

    @Override
    public ServletInputStream getInputStream() throws IOException {
        return new RequestInputStream(bytes);
    }
}
```
#### Response Wrapper ####
```
public class ResponseWrapper extends HttpServletResponseWrapper {

    protected ByteArrayOutputStream byteArrayOutputStream;

    public ResponseWrapper(HttpServletResponse response) {
        super(response);
        byteArrayOutputStream = new ByteArrayOutputStream();
    }

    public byte[] getByteArray() {
        return byteArrayOutputStream.toByteArray();
    }

    public String toString() {
        return byteArrayOutputStream.toString();
    }

    @Override
    public ServletOutputStream getOutputStream() throws IOException {
        return new ResponseOutputStream(super.getOutputStream(), byteArrayOutputStream);
    }
}
```
### Extending Streams - Caching request and response ###
#### ServletInputStream ####
```
public class RequestInputStream extends ServletInputStream{

    protected ByteArrayInputStream byteArrayInputStream;

    public RequestInputStream(byte[] bytes) {
        this.byteArrayInputStream = new ByteArrayInputStream(bytes);
    }

    @Override
    public int read() throws IOException {
        return this.byteArrayInputStream.read();
    }
}
```
#### ServletOutputStream ####
```
public class ResponseOutputStream extends ServletOutputStream {

    protected ServletOutputStream stream;
    protected ByteArrayOutputStream cache;

    public ResponseOutputStream(ServletOutputStream stream, ByteArrayOutputStream cache) {
        super();
        this.stream = stream;
        this.cache = cache;
    }

    @Override
    public void write(int b) throws IOException {
        checkNullStreams();
        stream.write(b);
        cache.write(b);
    }

    @Override
    public void write(byte[] b) throws IOException {
        checkNullStreams();
        stream.write(b);
        cache.write(b);
    }

    @Override
    public void write(byte[] b, int off, int len) throws IOException {
        checkNullStreams();
        stream.write(b, off, len);
        cache.write(b, off, len);
    }

    public ByteArrayOutputStream getBuffer() {
        return cache;
    }

    protected void checkNullStreams() throws IOException {
        if (stream == null)
            throw new IOException("ServletOutputStream stream: null, unable to write");
        else if (cache == null)
            throw new IOException("ByteArrayOutputStream cache: null, unable to write");
    }

}
```
### Interceptor ###
```
public class AuditInterceptor implements HandlerInterceptor{
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object o) throws Exception {
        RequestWrapper requestWrapper = (RequestWrapper) request;
        System.out.println(requestWrapper.toString());
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object o, ModelAndView modelAndView) throws Exception {
        //To change body of implemented methods use File | Settings | File Templates.
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object o, Exception e) throws Exception {
        ResponseWrapper responseWrapper = (ResponseWrapper) response;
        System.out.println(responseWrapper.toString());
    }
}
```
### web.xml ###
Make sure filter name `requestResponseWrapperFilter` matches with the filter bean. In our case I have annotated the bean with `@Component`, this will create a bean with the name `requestResponseWrapperFilter`
```
<!-- Request & Response Wrapper Filter -->
    <filter>
        <filter-name>requestResponseWrapperFilter</filter-name>
        <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    </filter>

    <filter-mapping>
        <filter-name>requestResponseWrapperFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
```
PS: Please feel free to submit a change. Thank you!
