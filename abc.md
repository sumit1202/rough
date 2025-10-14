```java
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.Objects;
import lombok.extern.slf4j.Slf4j;

/**
 * A lightweight wrapper that ensures every new connection is opened
 * with a freshly retrieved access token from the credential provider.
 */
@Slf4j
public class TokenAwareDataSource extends DelegatingDataSource {

    private final DatasourceCredentialProvider credentialProvider;
    private final boolean useAccessTokenProperty;

    public TokenAwareDataSource(DataSource targetDataSource,
                                DatasourceCredentialProvider credentialProvider,
                                boolean useAccessTokenProperty) {
        super(targetDataSource);
        this.credentialProvider = Objects.requireNonNull(credentialProvider);
        this.useAccessTokenProperty = useAccessTokenProperty;
    }

    @Override
    public Connection getConnection() throws SQLException {
        return prepareConnection(super.getConnection());
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return prepareConnection(super.getConnection(username, password));
    }

    private Connection prepareConnection(Connection conn) throws SQLException {
        try {
            String token = credentialProvider.getToken();
            if (useAccessTokenProperty) {
                // Denodo DVP expects "setClientInfo" for OAuth2 token
                conn.setClientInfo("accessToken", token);
                log.debug("🔐 Refreshed DVP access token on connection");
            } else {
                // For batch metadata case (password-based)
                conn.setClientInfo("password", token);
                log.debug("🔐 Refreshed batch metadata token on connection");
            }
        } catch (Exception e) {
            log.error("❌ Failed to refresh token on connection: {}", e.getMessage(), e);
            throw new SQLException("Failed to refresh access token", e);
        }
        return conn;
    }
}

```

```java
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.io.PrintWriter;
import java.util.logging.Logger;

public abstract class DelegatingDataSource implements DataSource {

    private final DataSource targetDataSource;

    protected DelegatingDataSource(DataSource targetDataSource) {
        this.targetDataSource = targetDataSource;
    }

    protected DataSource getTargetDataSource() {
        return targetDataSource;
    }

    @Override
    public Connection getConnection() throws SQLException {
        return targetDataSource.getConnection();
    }

    @Override
    public Connection getConnection(String username, String password) throws SQLException {
        return targetDataSource.getConnection(username, password);
    }

    @Override
    public <T> T unwrap(Class<T> iface) throws SQLException {
        return targetDataSource.unwrap(iface);
    }

    @Override
    public boolean isWrapperFor(Class<?> iface) throws SQLException {
        return targetDataSource.isWrapperFor(iface);
    }

    @Override
    public PrintWriter getLogWriter() throws SQLException {
        return targetDataSource.getLogWriter();
    }

    @Override
    public void setLogWriter(PrintWriter out) throws SQLException {
        targetDataSource.setLogWriter(out);
    }

    @Override
    public void setLoginTimeout(int seconds) throws SQLException {
        targetDataSource.setLoginTimeout(seconds);
    }

    @Override
    public int getLoginTimeout() throws SQLException {
        return targetDataSource.getLoginTimeout();
    }

    @Override
    public Logger getParentLogger() {
        try {
            return targetDataSource.getParentLogger();
        } catch (Exception e) {
            return Logger.getGlobal();
        }
    }
}

```


```java
@Configuration
public class DataSourceConfiguration {

    @Bean
    @Primary
    public DataSource batchMetadataDataSource(
            @Qualifier("batchMetadataDataSourceProperties") DataSourceProperties dataSourceProperties,
            @Qualifier("batchMetadataDataSourceCredentialProvider") DatasourceCredentialProvider credentialProvider,
            Environment environment) {

        HikariConfig config = new HikariConfig();
        config.setDriverClassName(dataSourceProperties.getDriverClassName());
        config.setJdbcUrl(dataSourceProperties.getUrl());
        config.setUsername(dataSourceProperties.getUsername());

        if (environment.acceptsProfiles(Profiles.of("local"))) {
            config.setPassword(dataSourceProperties.getPassword());
        } else {
            config.setPassword(credentialProvider.getToken());
        }

        HikariDataSource hikari = new HikariDataSource(config);
        return new TokenAwareDataSource(hikari, credentialProvider, false);
    }

    @Bean("dataVirtualizationPlatformDataSource")
    public DataSource dataVirtualizationPlatformDataSource(
            @Qualifier("dataVirtualizationPlatformDataSourceProperties") DataSourceProperties dataSourceProperties,
            @Qualifier("dataVirtualizationPlatformDataSourceCredentialProvider") DatasourceCredentialProvider credentialProvider,
            @Qualifier("dataVirtualizationPlatformDataSourceSslProperties") DataSourceSslConfigurationProperties ssl,
            Environment environment) {

        HikariConfig config = new HikariConfig();
        config.setDriverClassName(dataSourceProperties.getDriverClassName());
        config.setJdbcUrl(dataSourceProperties.getUrl());

        config.addDataSourceProperty("useOAuth2", true);
        config.addDataSourceProperty("accessToken", credentialProvider.getToken());

        if (!environment.acceptsProfiles(Profiles.of("local"))) {
            config.addDataSourceProperty("ssl", true);
            config.addDataSourceProperty("sslTrustStoreLocation", ssl.getKeyStorePath());
            config.addDataSourceProperty("sslTrustStorePassword", ssl.getKeyStorePassword());
        }

        HikariDataSource hikari = new HikariDataSource(config);
        return new TokenAwareDataSource(hikari, credentialProvider, true);
    }
}

```
