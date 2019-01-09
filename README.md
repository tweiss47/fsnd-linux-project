# Linux web server configuration project

## Server Information

|   |   |
| - | - |
| IP Address | 54.188.6.243 |
| User Name | grader |
| Catalog Application | http://54.188.6.243.xip.io/songcat/ |

## Configuration Steps

Here are the steps I followed to configure my application on Linux. Most of
the configuration steps were following the course _Deploying to Linux
Servers_ module and the _Getting started on Lightsail_ lesson.

### Creating a VM on Lightsail

This was almost all strait out of the lesson. The instance I created an
instance named _tweiss47-fsnd-02_. Actually I create one without the 02, but
I managed to [cough, cough] misspell my user name when configuring an ssh
whitelist. You can probably guess what happened then. So I got some practice
setting up snapshots the second time around.

There were a couple of additional steps like setting up the IAM account that
aren't really project relevant. I also added port 2200 to the Firewall on the
Networking tab at this time.

Finally I configured windows bash to allow me to ssh directly to using the
built in user by downloading the default key from the console. And copying
to `~/.ssh` and setting permissions on the private key file.

### Setting up the grader user

* Create the grader user
```
sudo adduser grader
```

* Enable sudo for grader
```
sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader
# replace ubuntu with grader
sudo sed -i 's/ubuntu/grader/g' /etc/sudoers.d/grader
```

* Enable ssh for grader
```
# on client
ssh-keygen -f .ssh/fsnd-grader-key
scp -i .ssh/<keyfile> .ssh/fsnd-grader-key.pub ubuntu@54.188.6.243:~

# as grader on remote server
mkdir .ssh
chmod 700 .ssh
sudo mv /home/ubuntu/fsnd-grader-key.pub .ssh/authorized_keys
sudo chown grader .ssh/authorized_keys
sudo chgrp grader .ssh/authorized_keys
chmod 600 .ssh/authorized_keys

# from client ssh as grader
ssh -i .ssh/fsnd-grader-key grader@54.188.6.243
```

### Lockdown SSH

The changes to lockdown SSH mostly involve adding directives to the SSH
configuration file `/etc/ssh/sshd_config`.

* Ensure there is no password based authentication allowed
```
PasswordAuthentication no
ChallengeResponseAuthentication no
```

* Ensure that the root user cannot log in remotely. For this step I followed
the advice on [Ask Ubuntu](https://askubuntu.com/questions/27559/how-do-i-disable-remote-ssh-login-as-root-from-a-server`)
```
PermitRootLogin no
DenyUsers root
AllowUsers ubuntu grader
```

* Restart the service to apply the new configuration
```
sudo service ssh restart

# now ssh requires -p 2200 parameter
ssh -i .ssh/fsnd-grader-key grader@54.188.6.243 -p 2200
```

* Configure ufw to only allow http (80) ssh (2200) and ntp (123)
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow www
sudo ufw allow 2200/tcp
sudo ufw allow ntp

# check configuration
sudo ufw show added
sudo ufw enable
sudo ufw status
```

### Install and configure apache

* Install apache
```
sudo apt-get install apache2
```

* Verify that http://54.188.6.243.xip.io/ returns the default ubuntu linux page

* Setup python
```
python3 --version
# returns Python 3.6.7
sudo apt install python3-pip
```

* Install mod_wsgi for python3
```
sudo apt-get install libapache2-mod-wsgi-py3
```

* Setup a hello world wsgi script at http://54.188.6.243.xip.io/hello to
check WSGI functionality.
```
sudo mkdir /var/www/hello
sudo vim /var/www/hello/hello.wsgi

# add script mapping to appache config
# WSGIScriptAlias /hello /var/www/hello/hello.wsgi
sudo vim /etc/apache2/sites-enabled/000-default.conf

sudo apache2ctl restart
```

### Install the catalog project (songcat) onto the server

My catalog project is located at
https://github.com/tweiss47/fsnd-catalog-project. These steps are mostly
covered in the README.md although I made a coulpe minor changes.

* Clone the project into the grader home directory.
```
git clone https://github.com/tweiss47/fsnd-catalog-project.git
```

* Install the project and all its dependencies. I decided to go ahead and
install globally although in retrospect it probably would have been better to
do in a virtual environment
```
cd fsnd-catalog-project
sudo -H pip3 install dist/songcat-0.1.1-py3-none-any.whl
```

* Inialize the database schema and setup test data
```
export FLASK_APP=songcat
flask init-model
flask init-test-data
```

* Allow apache to write to the database. I didn't reconfigure the project to
use postgres. It was built with sqllite so I went ahead and kept that
implementation.
```
chmod g+w instance/songcat.db
sudo chgrp www-data instance/songcat.db
sudo chgrp www-data instance
```

* Define a new SECRET_KEY for the deployment.
```
python3 -c 'import os; print(os.urandom(16))'
# write output to instance/config.py
# SECRET_KEY = b'<output>'
```

* Update the google auth [settings](https://console.developers.google.com) for
the new deployment. Add http://54.188.6.243.xip.io/ to the allowed origins.

### Configure apache to run songcat

* Add a WSGI script to initialize the application.
```
# in /var/www/songcat/songcat.wsgi
import sys
sys.path.insert(0, '/home/grader/fsnd-catalog-project')
from songcat import create_app
application = create_app()
```

* Add the script mapping to apache config and restart as with the hello test
script.

### Ensure all software is up to date

Run these update commands both before and after project setup to ensure that
all software on the server is up to date.
```
sudo apt-get update
sudo apt-get upgrade
```

## Final Thoughts

This was a fun project. I ended up running through the configuration a couple
of times between going through the course lessons and working on the project
VM.

All the procedures above were taken directly from the course lessons except
where noted either here or in the catalog project
[README](https://github.com/tweiss47/fsnd-catalog-project/blob/master/README.md).
And made use of the Amazon Lightsail
[documentation](https://lightsail.aws.amazon.com/ls/docs/en/overview)
to get SSH keys setup and other account features completed.
