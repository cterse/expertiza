# Deployment in Expertiza

This document provides steps for using Capistrano to deploy code to various environments specifed in `config/deploy` dir of the project root. 
</br>Also, steps for implementing automated dpeloyments on successful Travis builds are outlined in this document.

## Deployment Targets

| Expertiza Branch | Target Server | IP Address | Deployment User | Deployment Directory
|---|---|---|---|---|
| deployment_fix | VCL - Master branch testing server	| 152.7.98.236 | cterse | `/var/www` |

## Deployment Steps

Follow the following steps to configure a new server as a deployment target and start deployment using Capistrano. Most of the steps outlined under "Configuring a new Target Server 🎯" need to be run only once when configuring a new target server, and need not be run on consecutive deployments. 

### Configuring a new Target Server 🎯
Follow the following steps to setup a new deployment target server for both manual deployments using the `cap <env> deploy` command as well as automated Travis deployments.
1. Get access to a user account with sudo access.
2. Setup passwordless SSH access to this target from the machines you would want to deploy from.
3. Make sure the following are installed on the target:
  - `rvm`
  - JDK 8
  - git
  - `mysql-server`
  - `mysql-devel`
  - NodeJS
  - `npm`
  - `bundler` gem
4. Make sure the deployment user has MySQL remote login enabled with no password on both `localhost` and `127.0.0.1`. Use the following commands in the mysql terminal:
```sql
create user 'root'@'127.0.0.1' identified by '';
grant all privileges on *.* to 'root'@'127.0.0.1' with grant option;
flush privileges;
```
5. Run following commands to set relevant Java environment variables:
```bash
export JAVA_HOME=$(dirname $(dirname $(readlink $(readlink $(which javac)))))
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/jre/lib:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
```
6. Install required ruby versions using `rvm`.
7. Ensure `secrets.yml` and `database.yml` are present in `<project-root>/shared/config` dir on the target.</br>Ensure the correct password is specified in `database.yml`.</br>If the directory structure is not present on the target, initiate a deployment using `cap <env> deploy`. This deployment will fail, but will create the required dirrectory structure, in which you can place the above two files. Or, manually create the directory structure.
8. Ensure `public1.pem` and `private2.pem` are present in `<project-root>/shared` dir on the target. Refer point 6. if the directory structure is not present. 
9. Setup firewall access rules as follows:
```bash
sudo iptables -I INPUT -p tcp -s 0.0.0.0/0 --dport 8080 -j ACCEPT
sudo ufw allow 8080 (run again if it fails)
sudo ufw reload
```
10. For automated deployments from Travis servers, run command to allow Travis servers in firewall:
```bash
sudo iptables -I INPUT -p tcp -s "$(dig +short nat.travisci.net | tr -s '\r\n' ',' | sed -e 's/,$/\n/')" --dport 22 -j ACCEPT
```

**Hint:** To test your new configuration, manually clone the desired Expertiza branch in some random dir on the server, run `bundle install`, `rake db:migrate` and start the Rails server using `rails s`, all inside the cloned repo. If the application loads up correctly, the server is ready for remote deployments provided correct SSH setup.


## `/.travis.yml`

1. Change rvm to 2.6.6
2. Add branch you want to deploy under `branches`.
3. Add following section:
```yml
after_success:
- openssl aes-256-cbc -k $DEPLOY_KEY -in config/deploy_id_rsa_enc_travis -d -a -out config/deploy_id_rsa
- chmod 400 config/deploy_id_rsa_enc_travis
- chmod 400 config/deploy_id_rsa
- ssh-add -k config/deploy_id_rsa
- bundle exec cap staging deploy --trace
```

2. Add the following lines at the end:
```ruby
gem 'ed25519', '1.2.4'
gem 'bcrypt_pbkdf', '>= 1.0', '< 2.0'
```

## Capfile

Add `require 'capistrano/bower'` to Capfile to install all the npm dependencies defined in `/bower.json` during deployment.

## `/config/deploy.rb`

1. Change all occurrences of `production` to `staging`.
2. Edit line and set to `lock '~> 3.17.0'`
3. Edit line and set to `set :repo_url, 'https://github.com/<YOUR_GITHUB_USER>/expertiza.git'`
4. Edit line and set to `set :rvm_ruby_version, '2.6.6'`
5. Edit line and set to `set :deploy_to, "/home/<username>/expertiza_deploy"` E.g.:`/home/krshah3/expertiza_deploy"`
6. Edit line and set to `set :branch, 'deploy'`
7. Make sure `JAVA_HOME` under `set :default_env` is correctly set according to the value in the remote server.

**TIP:** Add `Rake::Task["deploy:migrate"].clear_actions` to `deploy.rb` to disable the migrate rake tasks during deployment. 

## `/config/deploy/staging.rb`

1. Edit and set line to `server '<YOUR_DEPLOYMENT_SERVER>', user: '<SERVER_USER>', roles: %w[web app db], my_property: :my_value`
2. Edit user name in following lines:
```ruby
role :app, %w[<SERVER_USER>@<YOUR_DEPLOYMENT_SERVER>]
role :web, %w[<SERVER_USER>@<YOUR_DEPLOYMENT_SERVER>]
role :db,  %w[<SERVER_USER>@<YOUR_DEPLOYMENT_SERVER>]
```

## Gemfile
1. Add `gem 'capistrano-bower'` to Gemfile, to install all the npm dependencies in the target server.
2. run `bundle lock --update` after above changes to generate a new Gemfile.lock.</br>
**ERROR:** `Could not find gem 'ruby (~> 2.3.1.0)' in the local ruby installation. The source contains 'ruby' at: 2.6.6.146` error. : Make sure you are on the right `deploy` branch.

## `/bower.json`
1. Add dependency `"tinymce": "latest"` in the bower.json file.

## Local machine

Follow the bewlo steps to encrypt the secret private key and upload it in the repository. This private key would be used to ssh and deploy into the target server.
```bash
gem instal travis
travis login --pro --github-token <token>
travis encrypt DEPLOY_KEY="password for encryption" --add
openssl aes-256-cbc -k "password for encryption" -in ~/.ssh/id_rsa -out deploy_id_rsa_enc_travis -a
```

Check for further reference: https://gist.github.com/waynegraham/5c6ab006862123398d07 .
