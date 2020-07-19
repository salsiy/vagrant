Vagrant.configure("2") do |config|

#nginx

  config.vm.define "nginx" do |nginx|

    nginx.vm.box = "debian/jessie64"
    nginx.vm.network "private_network", ip: "10.1.10.10"
    nginx.vm.hostname = "nginx"
    nginx.vm.provider :virtualbox do |v|

      v.customize ["modifyvm", :id, "--memory", "500"]
      end

    nginx.vm.provision "shell", inline: <<-SHELL

      sudo echo "10.1.10.11 wp1" | sudo tee -a /etc/hosts
      sudo echo "10.1.10.12 wp2" | sudo tee -a /etc/hosts
      sudo echo "10.1.10.13 db" | sudo tee -a /etc/hosts

      sudo apt-get install nginx --assume-yes
      sudo systemctl  start nginx
      sudo systemctl enable nginx
#configuring nginx as load balancer
      echo '#### 
      upstream backend {
        server 10.1.10.11;
        server 10.1.10.12;
    }
        
    server {
        listen      80 default_server;
        listen      [::]:80 default_server;

        location / {
                proxy_redirect      off;
                proxy_set_header    X-Real-IP \$remote_addr;
                proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
                proxy_set_header    Host \$http_host;
                proxy_pass http://backend;
        }
}
' > /etc/nginx/conf.d/load.conf
rm -rf /etc/nginx/sites-available/default
systemctl restart nginx

      
    
    SHELL
  end

#db
  config.vm.define "db" do |db|

    db.vm.box = "debian/jessie64"
    db.vm.network "private_network", ip: "10.1.10.13"
    db.vm.hostname = "db"
    db.vm.provider :virtualbox do |v|

      v.customize ["modifyvm", :id, "--memory", "500"]
      end

    db.vm.provision "shell", inline: <<-SHELL

      sudo echo "10.1.10.11 wp1" | sudo tee -a /etc/hosts
      sudo echo "10.1.10.12 wp2" | sudo tee -a /etc/hosts
#installmysql
      debconf-set-selections <<< 'mysql-server mysql-server/root_password password MySuperPassword'
      debconf-set-selections <<< 'mysql-server mysql-server/root_password_again password MySuperPassword'
      apt-get update
      apt-get install -y mysql-server
#configuredb
      rootpass="MySuperPassword"
      dbname="wordpress" 
      dbuser="admin"
      userpass="hellosamy"
      echo "CREATE DATABASE $dbname;" | mysql -u root -p$rootpass
      echo "CREATE USER '$dbuser'@'localhost' IDENTIFIED BY '$userpass';" | mysql -u root -p$rootpass
      echo "GRANT ALL PRIVILEGES ON $dbname.* TO '$dbuser'@'%';" | mysql -u root -p$rootpass
      echo "GRANT ALL PRIVILEGES ON  *.* TO 'root'@'%';" | mysql -u root -p$rootpass
      echo "GRANT ALL PRIVILEGES ON *.* TO 'root'@'wp1' IDENTIFIED BY 'MySuperPassword' with grant option;" | mysql -u root -p$rootpass
      echo "GRANT ALL PRIVILEGES ON *.* TO 'root'@'wp2' IDENTIFIED BY 'MySuperPassword' with grant option;" | mysql -u root -p$rootpass
      echo "FLUSH PRIVILEGES;" | mysql -u root -p$rootpass
      
#changingmysqlconfig
      echo "Updating mysql configs in /etc/mysql/my.cnf."
      sudo sed -i "s/.*bind-address.*/bind-address = 0.0.0.0/" /etc/mysql/my.cnf
      echo "Updated mysql bind address in /etc/mysql/my.cnf to 0.0.0.0 to allow external connections."
      sudo systemctl restart mysql
      sudo systemctl enable mysql
    
    SHELL
  end

#wordpress1

  config.vm.define "wp1" do |wp1|

    wp1.vm.box = "debian/jessie64"
    wp1.vm.network "private_network", ip: "10.1.10.11"
    wp1.vm.hostname = "wp1"
    wp1.vm.provider :virtualbox do |v|

      v.customize ["modifyvm", :id, "--memory", "500"]
      end

    wp1.vm.provision "shell", inline: <<-SHELL

      sudo echo "10.1.10.13 db" | sudo tee -a /etc/hosts
      apt install -y php5 php5-mysql php5-curl php5-gd unzip
      cd /tmp
      wget -c http://wordpress.org/latest.zip
      unzip -q latest.zip -d /var/www/html/
      chown -R www-data.www-data /var/www/html/wordpress
      chmod -R 755 /var/www/html/wordpress
      mkdir -p /var/www/html/wordpress/wp-content/uploads
      chown -R www-data.www-data /var/www/html/wordpress/wp-content/uploads
#configuring wordpress with db details
      cd /var/www/html/wordpress
      cp wp-config-sample.php wp-config.php
      perl -pi -e "s/database_name_here/wordpress/g" wp-config.php
      perl -pi -e "s/username_here/root/g" wp-config.php  
      perl -pi -e "s/password_here/MySuperPassword/g" wp-config.php
      perl -pi -e "s/localhost/db/g" wp-config.php
      systemctl restart apache2
#set WP salts
      perl -i -pe'
        BEGIN {
          @chars = ("a" .. "z", "A" .. "Z", 0 .. 9);
          push @chars, split //, "!@#$%^&*()-_ []{}<>~\`+=,.;:/?|";
          sub salt { join "", map $chars[ rand @chars ], 1 .. 64 }
        }
        s/put your unique phrase here/salt()/ge
      ' wp-config.php
#configuring apache virtualhost
      echo "#### 
      <VirtualHost *:80>
      ServerAdmin admin@example.com
      DocumentRoot /var/www/html/wordpress/ 
      <Directory /var/www/html/wordpress/>
        Options +FollowSymlinks
        AllowOverride All
        Require all granted
      </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      CustomLog ${APACHE_LOG_DIR}/access.log combined
      </VirtualHost>" > /etc/apache2/sites-available/wordpress.conf
      a2ensite wordpress.conf
      cd /etc/apache2/sites-available
      rm -rf 000-default.conf
      cp wordpress.conf 000-default.conf
      systemctl restart apache2

    
    SHELL
  end

#wordpress2
  config.vm.define "wp2" do |wp2|

    wp2.vm.box = "debian/jessie64"
    wp2.vm.network "private_network", ip: "10.1.10.12"
    wp2.vm.hostname = "wp2"
    wp2.vm.provider :virtualbox do |v|

      v.customize ["modifyvm", :id, "--memory", "500"]
      end

    wp2.vm.provision "shell", inline: <<-SHELL

      sudo echo "10.1.10.13 db" | sudo tee -a /etc/hosts
      apt install -y php5 php5-mysql php5-curl php5-gd unzip
      cd /tmp
      wget -c http://wordpress.org/latest.zip
      unzip -q latest.zip -d /var/www/html/
      chown -R www-data.www-data /var/www/html/wordpress
      chmod -R 755 /var/www/html/wordpress
      mkdir -p /var/www/html/wordpress/wp-content/uploads
      chown -R www-data.www-data /var/www/html/wordpress/wp-content/uploads
#configuring wordpress with db details
      cd /var/www/html/wordpress
      cp wp-config-sample.php wp-config.php
      perl -pi -e "s/database_name_here/wordpress/g" wp-config.php
      perl -pi -e "s/username_here/root/g" wp-config.php  
      perl -pi -e "s/password_here/MySuperPassword/g" wp-config.php
      perl -pi -e "s/localhost/db/g" wp-config.php
      systemctl restart apache2
#set WP salts
      perl -i -pe'
        BEGIN {
          @chars = ("a" .. "z", "A" .. "Z", 0 .. 9);
          push @chars, split //, "!@#$%^&*()-_ []{}<>~\`+=,.;:/?|";
          sub salt { join "", map $chars[ rand @chars ], 1 .. 64 }
        }
        s/put your unique phrase here/salt()/ge
      ' wp-config.php
#configuring apache virtualhost
      echo "#### 
      <VirtualHost *:80>
      ServerAdmin admin@example.com
      DocumentRoot /var/www/html/wordpress/ 
      <Directory /var/www/html/wordpress/>
        Options +FollowSymlinks
        AllowOverride All
        Require all granted
      </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      CustomLog ${APACHE_LOG_DIR}/access.log combined
      </VirtualHost>" > /etc/apache2/sites-available/wordpress.conf
      a2ensite wordpress.conf
      cd /etc/apache2/sites-available
      rm -rf 000-default.conf
      cp wordpress.conf 000-default.conf
      systemctl restart apache2
    
    SHELL
  end



end
 
