Deploying Django on AWS (Ubuntu)
======

The workflow is as follows:

1. Prepare AWS account and set up ALB
2. Provision the server using [Ansible](http://www.ansible.com/home)
3. Spin up an RDS database
4. Deploy your Django project from Git and serve it with this stack:
  * [Gunicorn](http://gunicorn.org/)
  * [Nginx](http://nginx.org/)
  * [Supervisord](http://supervisord.org/)
  * [Virtualenv](http://virtualenv.readthedocs.org/en/latest/)

This stack comes with useful logging for gunicorn, supervisord, and nginx.  It uses logrotate for managing logs.  You can also use this playbook to deploy multiple apps to the same server.

`*** You probably don't want to use this for live use, it's more about providing a quick dev enviroment for common stack components ***`


##2 minute quick start

First of all, prepare your working environment.
Install python3 system-wide (e.g. `yum install python3`)

Install virtualenv as user:
```
python3 -m pip install virtualenv --user
```
Clone this repo:
```
git clone https://github.com/aegrodin/techops_ansible.git
```
Go to cloned repository directory, create and enter virtual environment:
```
virtualenv env
source env/bin/activate
```
Install required modules, including ansible:
```
pip3 install -r requirements.txt
```
check you now have ansible 2.8:
```
ansible --version
```
Start by opening the `vars/common.yml` file and providing values for all the non-commented variables listed there.  The application will be deployed and owned by `application_user`.  You'll need to provide their password hash by running the following command:

`mkpasswd --method=sha-512`

Later if you log into the server, you can easily change to this user with `su - [application_user]`.  When you do this you'll notice the venv is auto-activated and you are automatically in your web app directory.

In the `hosts` file you can find ansible server name made with EC2 tag underneath `[webservers]`.

Now, run the playbook against your server with the following command:

`ansible-playbook playbook.yml --ask-vault-pass`

Should any task fail, you can make corrections and run again from that task:

`ansible-playbook playbook.yml  --start-at-task='my task name'`


#Assumptions

This project assumes your `application_name` variable is named the same as your Django project.

It's also assumed your project is set up as below.  Other variations are of course possible but you'll have to change the `virtualenv_path`, `git_root`, and `django_dir` variables accordingly.

```
./media
./static
./requirements.txt
./application_name
		./app1
		./app2
		manage.py
		./application_name
			settings.py
			wsgi.py
			...
```

Finally, all sensitive settings are enviroment variables which are exported via the postactivate script (in roles/web/templates).  The example below, from [2 scoops of Django](http://twoscoopspress.org/products/two-scoops-of-django-1-6) is a nice example of how you can configure your local/testing/live settings files.

```
def get_env_variable(var_name):
  """ Get the environment variable or return exception """
  try:
    return os.environ[var_name]
  except KeyError:
    error_msg = "Set the %s environment variable" % var_name
    raise ImproperlyConfigured(error_msg)

DATABASES = {
  'default': {
    'ENGINE': 'django.db.backends.postgresql_psycopg2',
    'NAME': get_env_variable('DB_NAME'),
    'USER': get_env_variable('DB_USER'),
    'PASSWORD': get_env_variable('DB_PASSWORD'),
    'HOST': get_env_variable('DB_HOST'), # Empty for localhost through domain sockets or '127.0.0.1' for localhost through TCP.
    'PORT': '', # Set to empty string for default.
},
SECRET_KEY = get_env_variable('DJANGO_SECRET_KEY')
```

##Variable management and Ansible Vault

Ansible Vault allows you to easily encrypt .yml files which you'll want to commit version control.  The vars directory is a good place to organise your encrypted variable files.

`ansible-vault encrypt foo.yml bar.yml baz.yml #to encrypt existing files`

`ansible-vault create foo.yml #to create a new encrypted file`

`ansible-vault edit foo.yml`

To run a playbook with encrypted files:

`ansible-playbook site.yml --ask-vault-pass`


##Ongoing deployments

Use tags to run a bespoke collection of tasks on your hosts

`ansible-playbook build/playbook.yml -i build/inventory/hosts  --tags='deploy'`

Some useful server commands:

`sudo supervisorctl restart application_name`

Restart nginx, postgres, etc

`sudo service restart nginx`


##AWS credentials

If you want to use this playbook to spin up your AWS servers you'll need to get the following details from your AWS account.

  * ACCESS_KEY_ID
  * SECRET_ACCESS_KEY

Next, if you haven't already, you'll need to generate a keypair and configure the security groups and subnets for your VPC, along with the Application Load Balancer.  Having done all this, you should go to the `vars` directory and fill in all your information, remembering to encrypt sensitive information.


##To do
Development, Staging, Live setups, Better lockdown etc


##Credits
I've borrowed a big part of the ansible code from this [repo](https://github.com/bee-keeper/aws-ansible-django-deployment)

