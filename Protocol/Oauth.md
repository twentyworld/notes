
# Oauth2

从实现的角度看待四种不同的模式：
```java
public interface OAuthClient extends OAuthTokenDecoder {
    /**
     * It is used for Resource Owner Password Credentials Grant.
     *
     * @param username username
     *
     * @param password password
     *
     * @return OAuth token
     *
     * @throws IOException in case of a problem with connection
     *
     * @throws OAuthHttpResponseException represents unwanted http response codes
     *
     * @throws OAuthClientProtocolException in case of an protocol error
     */
    Token requestTokenForUser(String username, String password) throws IOException, OAuthHttpResponseException, OAuthClientProtocolException;

    /**
     * It is used for Client Credentials Grant. It uses default client id and client secret which may be specified for instance in constructor.
     *
     * @return OAuth token
     *
     * @throws IOException in case of a problem with connection
     *
     * @throws OAuthHttpResponseException represents unwanted http response codes
     *
     * @throws OAuthClientProtocolException in case of an protocol error
     */
    Token requestTokenForClient() throws IOException, OAuthHttpResponseException, OAuthClientProtocolException;

    /**
     * It is used for Client Credentials Grant.
     *
     * @param clientId client id
     *
     * @param clientSecret client Secret
     *
     * @return OAuth token
     *
     * @throws IOException in case of a problem with connection
     *
     * @throws OAuthHttpResponseException represents unwanted http response codes
     *
     * @throws OAuthClientProtocolException in case of an protocol error
     */
    Token requestTokenForClient(String clientId, String clientSecret) throws IOException, OAuthHttpResponseException, OAuthClientProtocolException;

    /**
     * It returns authorization code URI used in Authorization Code Grant. Server should redirect browser client to this URI.
     *
     * @param request request URI received by server.
     *
     * @param callback oauth callback which will receive authorization code
     *
     * @return Authorization code URI
     */
    URI getAuthCodeUri(URI request, URI callback);

    /**
     * It is used for Authorization Code Grant.
     *
     * @param authCode authorization code
     *
     * @param callback the same callback as in {@link #getAuthCodeUri(URI, URI) getAuthCodeUri}, it is for extra check
     *
     * @return OAuth token.
     *
     * @throws IOException in case of a problem with connection
     *
     * @throws OAuthHttpResponseException represents unwanted http response codes
     *
     * @throws OAuthClientProtocolException in case of an protocol error
     */
    Token requestTokenForCode(String authCode, URI callback) throws IOException, OAuthHttpResponseException, OAuthClientProtocolException;

    /**
     * It returns logout URI. Server should redirect browser client to this URI.
     *
     * @param logoutRedirection OAuth server will redirect browser client to this URI
     *
     * @return OAuth server URI used for logout
     */
    URI getLogoutUri(URI logoutRedirection);

}

```
具体实现的类：

```java

public class OAuthClientImpl implements OAuthClient {

    private static final Logger logger = LoggerFactory.getLogger(OAuthClientImpl.class);
    private final OAuthClientConfig config;
    private final OAuthTokenDecoder oAuthTokenDecoder;
    private final OAuthHttpClient httpClient;

    public OAuthClientImpl(OAuthClientConfig config) {
        this(config, new OAuthHttpClient(config));
    }

    OAuthClientImpl(OAuthClientConfig config, OAuthHttpClient httpClient) {
        notNull(config.getAuthServer());
        notNull(config.getClientId());
        notNull(config.getClientSecret());
        notNull(config.getUserAgent());
        this.config = config;
        this.httpClient = httpClient;
        this.oAuthTokenDecoder = config.getTokenDecoder() != null ? config.getTokenDecoder() : new OAuthTokenDecoderImpl(config, httpClient);
    }

    @Override
    public Token requestTokenForUser(String username, String password) throws IOException {
        logger.debug("Request token for user {}", username);

        UrlEncodedFormEntity formEntity = new UrlEncodedFormEntity(
                appendCommonParameters(
                        new BasicNameValuePair("grant_type", "password"),
                        new BasicNameValuePair("username", username),
                        new BasicNameValuePair("password", password)),
                Consts.UTF_8);

        URIBuilder builder = uriBuilder(config.getAuthServer()).setPath(config.getTokenEndpoint());
        HttpPost httpPost = new HttpPost(toURI(builder));
        httpPost.setEntity(formEntity);
        Map<String, ?> response = httpClient.executeRequest(httpPost, config.getClientId(), config.getClientSecret());
        String token = (String) response.get("access_token");
        return decode(token);
    }

    @Override
    public Token requestTokenForClient() throws IOException {
        return requestTokenForClient(config.getClientId(), config.getClientSecret());
    }

    @Override
    public Token requestTokenForClient(String clientId, String clientSecret) throws IOException {
        logger.debug("Request token for clientId {}", clientId);

        UrlEncodedFormEntity formEntity = new UrlEncodedFormEntity(
                appendCommonParameters(new BasicNameValuePair("grant_type", "client_credentials")),
                Consts.UTF_8);

        URIBuilder builder = uriBuilder(config.getAuthServer()).setPath(config.getTokenEndpoint());
        HttpPost httpPost = new HttpPost(toURI(builder));
        httpPost.setEntity(formEntity);
        Map<String, ?> response = httpClient.executeRequest(httpPost, clientId, clientSecret);
        String token = (String) response.get("access_token");
        return decode(token);
    }

    private List<BasicNameValuePair> appendCommonParameters(BasicNameValuePair... params) {
        ArrayList<BasicNameValuePair> parameters = new ArrayList<BasicNameValuePair>(Arrays.asList(params));
        parameters.add(new BasicNameValuePair("client_id", config.getClientId()));
        parameters.add(new BasicNameValuePair("client_secret", config.getClientSecret()));
        if (config.getTokenScope() != null && config.getTokenScope().length() > 0) {
            parameters.add(new BasicNameValuePair("scope", config.getTokenScope()));
        }
        return parameters;
    }

    @Override
    public URI getAuthCodeUri(URI request, URI callback) {
        return toURI(uriBuilder(config.getAuthServer())
                .setPath(config.getAuthorizeEndpoint())
                .setParameter("response_type", "code")
                .setParameter("client_id", config.getClientId())
                .setParameter("redirect_uri", callback.toString())
                .setParameter("state", request.toString()));
    }

    @Override
    public Token requestTokenForCode(String authCode, URI callback) throws IOException {
        logger.debug("Fetch token for code {}", authCode);

        UrlEncodedFormEntity formEntity = new UrlEncodedFormEntity(Arrays.asList(
                new BasicNameValuePair("grant_type", "authorization_code"),
                new BasicNameValuePair("code", authCode),
                new BasicNameValuePair("redirect_uri", callback.toString()),
                new BasicNameValuePair("client_id", config.getClientId()),
                new BasicNameValuePair("client_secret", config.getClientSecret())),
                Consts.UTF_8);

        URIBuilder builder = uriBuilder(config.getAuthServer()).setPath(config.getTokenEndpoint());
        HttpPost httpPost = new HttpPost(toURI(builder));
        httpPost.setEntity(formEntity);
        Map<String, Object> response = httpClient.executeRequest(httpPost, config.getClientId(), config.getClientSecret());
        String token = (String) response.get("access_token");
        return decode(token);
    }

    @Override
    public URI getLogoutUri(URI logoutRedirection) {
        return toURI(uriBuilder(config.getAuthServer())
                .setPath(config.getLogoutEndpoint())
                .setParameter("redirect_uri", logoutRedirection.toString()));
    }

    @Override
    public Token decode(String token) throws IOException {
        return oAuthTokenDecoder.decode(token);
    }

}

```
