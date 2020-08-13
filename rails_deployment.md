# Rails Deployment to AWS EC2
####  Stack : (Nginx + Phusion Passenger + Postgresql)
## By [Manoj Naidu](https://github.com/manojnaidu619)

#### [Here is the Video Tutorial](https://youtu.be/F0FsgktWf34)

* ### Step -1
  To **Launch an EC2** instance from aws console with all the credentials and configurations hooked.

* ### Step-2
  **Create User**, for application deployment. and assigning privileges.
  ``` bash
  $ sudo adduser ______ 		# Fill the Blank with username 
  ``` 
  
   ```bash
  $ sudo mkdir -p ~_____/.ssh  	# Fill the Blank with username 
  $ touch $HOME/.ssh/authorized_keys
  $ sudo sh -c "cat $HOME/.ssh/authorized_keys >> ~_____/.ssh/authorized_keys"  	# Fill the Blank with username 
  $ sudo chown -R _____: ~_____/.ssh    # Fill the Blank with username
  $ sudo chmod 700 ~_____/.ssh      # Fill the Blank with username
  $ sudo sh -c "chmod 600 ~myappuser/.ssh/*"    # Fill the Blank with username
    ```
    **Now, SSH into server under the username created now and follow next steps**
    
 * ### Step-3
   **Installing Dependencies**
    ```bash
     # Adding Node.js 10 repository
     $ curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
     
     # Adding Yarn repository
     $ curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
      
     $ echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
     
     $ sudo add-apt-repository ppa:chris-lea/redis-server
     
     # Refresh our packages list with the new repositories
     $ sudo apt-get update
      
     # Install our dependencies for compiiling Ruby along with Node.js and Yarn
     $ sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev dirmngr gnupg apt-transport-https ca-certificates redis-server redis-tools nodejs yarn
    ```
  * ### Step-4
    **Installing Rbenv and Ruby**
   ```bash
     $ git clone https://github.com/rbenv/rbenv.git ~/.rbenv
     $ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
     $ echo 'eval "$(rbenv init -)"' >> ~/.bashrc
     $ git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
     $ echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
     $ git clone https://github.com/rbenv/rbenv-vars.git ~/.rbenv/plugins/rbenv-vars
     $ exec $SHELL
     $ rbenv install 2.6.5        
     $ rbenv global 2.6.5
     $ ruby -v
     # ruby 2.6.5
   ```
   ```bash
     # This installs the latest Bundler, currently 2.x.
     $ gem install bundler
     # For older apps that require Bundler 1.x, you can install it as well.
     $ gem install bundler -v 1.17.3
     # Test and make sure bundler is installed correctly, you should see a version number.
     $ bundle -v
     # Bundler version 2.0
   ```
   * ### Step-5
     **Installing Nginx and Passenger**
       ```bash
         $ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
         $ sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger bionic main > /etc/apt/sources.list.d/passenger.list'
         $ sudo apt-get update
         $ sudo apt-get install -y nginx-extras libnginx-mod-http-passenger
         $ if [ ! -f /etc/nginx/modules-enabled/50-mod-http-passenger.conf ]; then sudo ln -s /usr/share/nginx/modules-available/mod-http-passenger.load /etc/nginx/modules-enabled/50-mod-http-passenger.conf ; fi
         $ sudo ls /etc/nginx/conf.d/mod-http-passenger.conf
       ```
      Now, edit the configuration file to point to correct Ruby version
      * Open the file
      `$ sudo nano /etc/nginx/conf.d/mod-http-passenger.conf`
      * edit, the line containing `passenger_ruby` to..
      `passenger_ruby /home/deploy/.rbenv/shims/ruby;`
      * Start the server
      `$ sudo service nginx start`
      
      Before continuing, need to edit the nginx conf file so..
      * Remove the old file created by nginx, by default
      `$ sudo rm /etc/nginx/sites-enabled/default`
      * Create own conf file..
      Note : change `myapp` to your app name
      `$ sudo nano /etc/nginx/sites-enabled/myapp` 
      * Add the following contents inside it..
      ```
      server {
        listen 80;
        listen [::]:80;

        server_name _;
        root /home/deploy/myapp/current/public;   # Change myapp to your appname

        passenger_enabled on;
        passenger_app_env production;

        location /cable {
         passenger_app_group_name myapp_websocket;
         passenger_force_max_concurrent_requests_per_process 0;
        }

        # Allow uploads up to 100MB in size
        client_max_body_size 100m;

        location ~ ^/(assets|packs) {
         expires max;
         gzip_static on;
        }
       }
      ```
      * Reload the Server now..
      `$ sudo service nginx reload`
  
   * ### Step-6
     **Installing and Creating Database**
     **Note**: 
          * change `deploy` to user-owner in line 3
          * change `myapp` to your database-name in line 4
     ```bash
      $ sudo apt-get install postgresql postgresql-contrib libpq-dev
      $ sudo su - postgres
      $ createuser --pwprompt deploy
      $ createdb -O deploy myapp
      $ exit
     ```
     Database can be accessed anytime by typing..
     
     `$ psql -U USERNAME -W -h 127.0.0.1 -d DBNAME`
    
* ### Step-7 (To be performed in local machine)
  **Setup Capistrano**
  * Add these to your Gemfile
  ```ruby
   gem 'capistrano', '~> 3.11'
   gem 'capistrano-rails', '~> 1.4'
   gem 'capistrano-passenger', '~> 0.2.0'
   gem 'capistrano-rbenv', '~> 2.1', '>= 2.1.4'
  ``` 
  * run these in terminal
  ```bash
  bundle install
  bundle exec cap install STAGES=production
  ```
  * The above command creates 3 files namely
  `Capfile`
  `config/deploy.rb`
  `config/deploy/production.rb`

    Now, editing them one by one..

    -> Inside `Capfile` make sure the below lines are uncommented/added.
    ```ruby
    require 'capistrano/rails'
    require 'capistrano/passenger'
    require 'capistrano/rbenv'

    set :rbenv_type, :user
    set :rbenv_ruby, '2.6.5'       # Spcify your ruby version 
    ```
    -> Inside `config/deploy.rb`, need to link repo and configs.
    ```ruby
     set :application, "myapp"
     set :repo_url, "git@github.com:username/MYAPP.git"

     # Deploy to the user's home directory
     set :deploy_to, "/home/deploy/#{fetch :application}"

     append :linked_dirs, 'log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle', '.bundle', 'public/system', 'public/uploads'

     # Only keep the last 5 releases to save disk space
     set :keep_releases, 5

     # Optionally, you can symlink your database.yml and/or secrets.yml file from the shared directory during deploy
     # This is useful if you don't want to use ENV variables
     # append :linked_files, 'config/database.yml', 'config/secrets.yml'
    ```
    -> Inside `config/deploy/production.rb`
     * Replace `1.2.3.4` with your server publice IP address
     * change `user: 'deploy'` to `user: 'YOUR_USERNAME'`

    Save these changes and now ssh into server and make some changes there...
    
* ### Step-8 (to be performed in server)  
  Now, lets create a file to hold all `ENV` variables
  ```bash
   $ mkdir /home/deploy/YOUR_APP_NAME
   $ sudo nano /home/deploy/YOUR_APP_NAME/.rbenv-vars
  ```
  Here, you can store environment variables like `SECRET_BASE_KEY`, `STRIPE_KEY`...
    
* ### Step-9 (to be performed in local machine)
  All Setup done, let's run final command..
  `$ cap production deploy`

  **NOTE**: if capistrano throws any error saying missing key/ secret key then..
   * ssh into sever
   * run..
   ```bash
     $ cd /home/deploy
     $ sudo nano ~/.bashrc
   ```
   * now add `SECRET_BASE_KEY` at the top of the file.
      
     
* ### Step-10 (to be performed in server) 
      $ sudo service nginx restart

## YOU MADE IT ! Your app is now live  
  [Follow me!](https://github.com/manojnaidu619)