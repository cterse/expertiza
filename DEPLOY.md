# Deployment in Expertiza

This document provides steps for using Capistrano to deploy code to various environments specifed in `config/deploy` dir of the project root. 
</br>Also, steps for implementing automated deployments on successful Travis builds are outlined in this document.

## Deployment Targets 🎯

| Expertiza Branch | Capistrano Environment | Target Server | IP Address | Deployment User | Deployment Directory
|---|---|---|---|---|---|
| deployment_fix | testing | VCL - Master branch testing server	| 152.7.98.236 | cterse | `/var/www` |
| main | testing | lin-res103.csc.ncsu.edu	| 152.14.92.215 | ??? | `/var/www` |
| legacy | production | lin-res44.csc.ncsu.edu	| 152.14.92.5 | expertiza | `/var/www` |

## Configuring a new Target Server 🎯
Follow the following steps to set up a new deployment target server for both manual deployments using the `cap <env> deploy` command as well as automated Travis deployments.
1. Get access to a user account with sudo access.
2. Set up passwordless SSH access to this target from the machines you would want to deploy from.
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
7. Ensure `secrets.yml` and `database.yml` are present in `<project-root>/shared/config` dir on the target.</br>Ensure the correct password is specified in `database.yml`.</br>If the directory structure is not present on the target, initiate a deployment using `cap <env> deploy`. This deployment will fail, but will create the required directory structure, in which you can place the above two files. Or, manually create the directory structure.
8. Ensure `public1.pem` and `private2.pem` are present in `<project-root>/shared` dir on the target. Run `cap <env> deploy` or manually create the directories if the directory structure is not present. 
9. Set up firewall access rules as follows:
```bash
sudo iptables -I INPUT -p tcp -s 0.0.0.0/0 --dport 8080 -j ACCEPT
sudo ufw allow 8080 (run again if it fails)
sudo ufw reload
```
10. For automated deployments from Travis servers, run command to allow Travis servers in firewall:
```bash
sudo iptables -I INPUT -p tcp -s "$(dig +short nat.travisci.net | tr -s '\r\n' ',' | sed -e 's/,$/\n/')" --dport 22 -j ACCEPT
```

**Hint:** To test your new configuration, manually clone the desired Expertiza branch in some random dir on the server, run `bundle install`, `rake db:migrate` and start the Rails server using `rails s`, all inside the cloned repo. If the application loads up correctly, the server is ready for remote deployments provided correct SSH set up.

## Deployment using Capistrano ⚙️
### Capfile
Capfile should be in the project root. Make sure the Capfile has `require 'capistrano/bower'` entry and that it does NOT have the `require 'capistrano/rails/migrations'` entry.

### Capistrano env files
Under the `config/deploy` dir are the various environment configuration files. For every env file, make sure that:
1. The targer server domain name/IP address is correct in all the places. 
2. The deployment user is correct and has passwordless SSH and sudo access on the target server.
3. Set the Java env: `set :default_env, 'JAVA_HOME' => '/usr/lib/jvm/java-8-oracle'`
4. Set the branch to be deployed in this envrionment: `set :branch, 'deployment_fix'`
5. Set the correct Ruby version (and make sure this version is installed in the target RVM): `set :rvm_ruby_version, '2.3.1'`


## Deployment using TravisCI 🤖
Travis CI is not a replacement for Capistrano. Travis just provides build validation. On a successful build, we can configure the Travis servers to deploy the build using Capistrano.

### Local machine
Follow the below steps to encrypt the secret private key and upload it in the repository. This private key would be used to ssh and deploy into the target server.
```bash
gem install travis
travis login --pro --github-token <token>
travis encrypt DEPLOY_KEY="password for encryption" --add
openssl aes-256-cbc -k "password for encryption" -in ~/.ssh/id_rsa -out deploy_id_rsa_enc_travis -a
```

### `/.travis.yml`
1. Add branch you want to deploy under `branches`.
2. Add following section:
```yml
after_success:
- openssl aes-256-cbc -k $DEPLOY_KEY -in config/deploy_id_rsa_enc_travis -d -a -out config/deploy_id_rsa
- chmod 400 config/deploy_id_rsa_enc_travis
- chmod 400 config/deploy_id_rsa
- ssh-add -k config/deploy_id_rsa
- bundle exec cap staging deploy --trace
```

### Gemfile
Add the following lines at the end:
```ruby
gem 'ed25519', '1.2.4'
gem 'bcrypt_pbkdf', '>= 1.0', '< 2.0'
```

### `/config/deploy/<env>.rb`
We have used staging.rb env as a testing env for Travis deployments. Make the following changes in the approproate Capistrano env file.
1. Edit and set line to `server '<YOUR_DEPLOYMENT_SERVER>', user: '<SERVER_USER>', roles: %w[web app db], my_property: :my_value`
2. Edit user name in following lines:
```ruby
role :app, %w[<SERVER_USER>@<YOUR_DEPLOYMENT_SERVER>]
role :web, %w[<SERVER_USER>@<YOUR_DEPLOYMENT_SERVER>]
role :db,  %w[<SERVER_USER>@<YOUR_DEPLOYMENT_SERVER>]
```

### `/bower.json`
1. Add dependency `"tinymce": "latest"` in the bower.json file. (only if there are errors)

Check for further reference: https://gist.github.com/waynegraham/5c6ab006862123398d07.

## Tips 💡
1. Add `Rake::Task["deploy:migrate"].clear_actions` to `deploy.rb` to disable the migrate rake tasks during deployment. 
2. Run `bundle lock --update` to update the `Gemfile.lock` after any changes to the `Gemfile`. Make sure to track both files in git.
3. VCL VMs may keep on crashing. Try a soft then a hard reboot if there are connection issues.
4. The expertiza.ncsu.edu (Production) server runs RHEL and the Master Testing VCL Server runs Ubuntu 16.04 -> good to know.
5. If you're testing a deployment with all success messages, no visible errors, but that does not load the final application, try accessing the application on the target server itself first. Use `curl` with the localhost as domain and appropriate port. If it works here, there are firewall issues. 
6. Travis CI charges as per the time required for builds. For testing, disable build time-consuming or already-tested components under the `matrix` section in `.travis.yml` 
