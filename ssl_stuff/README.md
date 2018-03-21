For client-based authentication to work, postgresql requires the following files:
* `server.crt` and `server.key` for the Postgres server.
* `client.crt` and `client.key` for the connecting client
* `root.crt` for the certificate authority -- this is what's used to verify the client's certificate for validity (`client.crt`). Note that we cannot enable certificate based authentication (i.e. use `client.crt`) without a certificate authority per https://github.com/postgres/postgres/blob/master/src/backend/libpq/auth.c#L375-L384 (any `client.crt` will always be reported as an invalid certificate).

In the following discussion, let `PGDATA` be the postgres data directory. On RHEL 7, this is `/var/lib/pgsql/data`. For simplicity, we will be using the `server.crt` file as our certificate authority to create our `client.crt` file.

# Enable SSL mode on POSTGRES
* Create the `server.crt`, `server.key`, and the `root.crt` files. These are done as follows (written in a bash script style for clarity):
```
pushd $PGDATA
  # 'genrsa' generates server.key but requires a passphrase.
  # `rsa` removes that entered passphrase
  openssl genrsa -des3 -out server.key 1024
  openssl rsa -in server.key -out server.key

  # Creates a new request using our generated server.key file. Key is valid for 3650 days.
  # The CN is the DB host's fqdn. The -x509 option indicates that our request is self-signed
  # (since we are using `server.crt` as the CA). Note we can configure the # of days and also
  # add other metadata in the -subj part.
  openssl req -new -key server.key -days 3650 -out server.crt -x509 -subj '/CN=r5w6h4gfiflybfs.delivery.puppetlabs.net

  # Create 'root.crt'
  cp server.crt root.crt
  
  # This is so postgres is able to start-up with the provided
  # 'server.crt' and 'server.key' files.
  chmod 600 server.key
  chown postgres server.key
  chgrp postgres server.key
popd
```
* Add the lines `ssl = on` and `ssl_ca_file = 'root.crt'` to `$PGDATA/postgresql.conf`.
^ Note the `ssl_ca_file` is the path to our certificate authority.

* Add the following entry to `${PGDATA}/pg_hba.conf`
```
hostssl <database> <user> <address> cert clientcert=1
```

For example with localhost, and for any user and DB, we would enter:
```
hostssl all all 127.0.0.1/32 cert clientcert=1
```

NOTE: The `clientcert=1` part forces the client to supply a certificate file for every connection. Also if you do not want ssl to be used for certain connections, you can specify those using `host` (or `local`) instead where appropriate.

* Restart the postgres server. On RHEL 7, you can run `systemctl restart postgresql`

# Creating the client certificate and key
* Do the following steps (psuedo-bash script style for clarity). Assume all operations are on the client-side host.
```
pushd SSL_DIR
  # Copy over 'root.crt' and `server.key` on the DB host to the client-side host.
  # Recall that `root.crt` is our CA; we need it to sign our client's CSR to create
  # the `client.crt` file.
  scp DB_HOST:$PGDATA/root.crt .
  scp DB_HOST:$PGDATA/server.key .


  # Create the client key
  openssl genrsa -des3 -out client.key 1024
  openssl rsa -in client.key -out client.key
  chmod 600 client.key

  # Create the client-side CSR and sign it. Note the CN entry is important, by default
  # it MUST be our database username. However we can change it to something else and add
  # an appropriate mapping if need be -- can specify these steps separately.
  openssl req -new -key client.key -out client.csr -subj '/CN=razor'
  openssl x509 -req -in client.csr -CA root.crt -CAkey server.key -out client.crt -CAcreateserial

  # Convert the client.key to pk8 format; this is so that JDBC can read it (see https://basildoncoder.com/blog/postgresql-jdbc-client-certificates.html)
  # -nocrypt is necessary to avoid a password prompt on every connection
  openssl pkcs8 -topk8 -nocrypt -inform PEM -outform DER -in client.key -out client.pk8
popd
```

# Connecting
After all of the preceding steps are completed, we can connect to our DB from the client via. 
```
Sequel.connect('jdbc:postgresql://r5w6h4gfiflybfs.delivery.puppetlabs.net:5432/razordb?user=razor&sslmode=require&sslcert=SSL_DIR/client.crt&sslkey=SSL_DIR/client.pk8')
```

We can also achieve the same thing with:
```
Sequel.connect(
  'jdbc:postgresql://r5w6h4gfiflybfs.delivery.puppetlabs.net:5432/razordb?user=razor',
  :jdbc_properties => {
    :sslmode => "require",
    :sslcert => "SSL_DIR/client.crt",
    :sslkey => "SSL_DIR/client.pk8"
  }
)
```

NOTE: The DB user should be passed in as part of the URL. Otherwise, postgres will name the current client-side system user as the connecting DB user, which can cause authentication to fail and the failure difficult to track.
