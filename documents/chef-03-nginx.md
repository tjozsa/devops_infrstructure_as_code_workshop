# Deploying Nginx

1. First we have to create a folder called cookbooks where we will go to insert our cookbooks:
    ```
    mkdir cookbooks
    ```
2. Inside the cookbooks folder we create our first cookbook named mynginx:
    ```
    cd cookbooks
    chef generate cookbook mynginx
    ```
3. Edit the file default.rb in in `mynginx/recipes/default.rb`:

    ```ruby
    package 'git'
    package 'tree'

    package 'nginx' do
        action :install
    end


    service 'nginx' do
        action [ :enable, :start ]
    end


    cookbook_file "/var/www/html/index.html" do
        source "index.html"
        mode "0644"
    end
    template "/etc/nginx/nginx.conf" do   
        source "nginx.conf.erb"
        notifies :reload, "service[nginx]"
    end
    ```
4. From inside the mynginx folder I must generate a file index.html:
    ```
    chef generate file index.html
    ```
5. Edit the file `index.html` in `files/default/index.html`:
    ```html
    <html>
    <head>
        <title>Hello there</title>
    </head>
    <body>
        <h1>This is a test</h1>
        <p>Please work!</p>
    </body>
    </html>
    ```
6. Create a templates named nginx.conf:
    ```
    chef generate template nginx.conf
    ```
7. Edit the file `nginx.conf.erb` in `templates/nginx.conf.erb` and insert your custom configuration of nginx.
    ```ruby
    user www-data;
    worker_processes auto;
    pid /run/nginx.pid;
    events {
        worker_connections 768;
        # multi_accept on;
    }
    http {
        ##
        # Basic Settings
        ##
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;
        # server_tokens off;
        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        ##
        # SSL Settings
        ##
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;
        ##
        # Logging Settings
        ##
        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;
        ##
        # Gzip Settings
        ##
        gzip on;
        gzip_disable "msie6";
        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
        ##
        # Virtual Host Configs
        ##
        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
    }
    #mail {
    #	# See sample authentication script at:
    #	# http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
    # 
    #	# auth_http localhost/auth.php;
    #	# pop3_capabilities "TOP" "USER";
    #	# imap_capabilities "IMAP4rev1" "UIDPLUS";
    # 
    #	server {
    #		listen     localhost:110;
    #		protocol   pop3;
    #		proxy      on;
    #	}
    # 
    #	server {
    #		listen     localhost:143;
    #		protocol   imap;
    #		proxy      on;
    #	}
    #}
    ```
8. Now you can launch `chef-client` in local mode for install or update your machine:
    ```
    sudo chef-client -z --runlist "mynginx"
    ```


Source: https://medium.com/@pierangelo1982/a-basic-nginx-cookbook-for-chef-ba95d801dbf3