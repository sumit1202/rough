```java
package com.example.datasource;

import com.example.security.DataSourceCredentialProvider;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.sql.DataSource;
import java.io.PrintWriter;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;
import java.util.Objects;
import java.util.Properties;
import java.util.logging.Logger as JULLogger;

public class DynamicTokenDataSource implements DataSource {

    private static final Logger log = LoggerFactory.getLogger(DynamicTokenDataSource.class);

    private final String jdbcUrl;
    private final String driverClassName;
    private final String username;
    private final DataSourceCredentialProvider credentialProvider;
    private final boolean useAccessToken;

    public DynamicTokenDataSource(String jdbcUrl,
                                  String driverClassName,
                                  String username,
                                  DataSourceCredentialProvider credentialProvider,
                                  boolean useAccessToken) {
        this.jdbcUrl = Objects.requireNonNull(jdbcUrl);
        this.driverClassName = Objects.requireNonNull(driverClassName);
        this.username = username;
        this.credentialProvider = Objects.requireNonNull(credentialProvider);
        this.useAccessToken = useAccessToken;

        try {
            Class.forName(driverClassName);
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException("❌ JDBC driver class not found: " + driverClassName, e);
        }
    }

    @Override
    public Connection getConnection() throws SQLException {
        String token = credentialProvider.getToken();
        Properties props = new Properties();
        if (username != null) props.setProperty("user", username);
        if (useAccessToken) {
            props.setProperty("accessToken", token);
        } else {
            props.setProperty("password", token);
        }

        try {
            Connection conn = DriverManager.getConnection(jdbcUrl, props);
            log.debug("✅ Created new connection for [{}]", useAccessToken ? "Denodo DVP" : "Batch Metadata");
            return conn;
        } catch (SQLException e) {
            if (isTokenExpiredError(e)) {
                log.warn("⚠️ Token expired — retrying with fresh token...");
                String newToken = credentialProvider.refreshToken();
                if (useAccessToken)
                    props.setProperty("accessToken", newToken);
                else
                    props.setProperty("password", newToken);
                return DriverManager.getConnection(jdbcUrl, props);
            }
            throw e;
        }
    }

    private boolean isTokenExpiredError(SQLException e) {
        String msg = e.getMessage();
        return msg != null && (msg.contains("expired") || msg.contains("invalid token"));
    }

    @Override public Connection getConnection(String username, String password) throws SQLException {
        return getConnection();
    }
    @Override public <T> T unwrap(Class<T> iface) { throw new UnsupportedOperationException(); }
    @Override public boolean isWrapperFor(Class<?> iface) { return false; }
    @Override public PrintWriter getLogWriter() { return null; }
    @Override public void setLogWriter(PrintWriter out) {}
    @Override public void setLoginTimeout(int seconds) {}
    @Override public int getLoginTimeout() { return 0; }
    @Override public JULLogger getParentLogger() { return JULLogger.getGlobal(); }
}


```
