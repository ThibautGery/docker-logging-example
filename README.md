Docker-logging-example
======================

Logging example in a docker architecture with fluentd and elasticsearch

Requirements
------------

 * [Vagrant](https://www.vagrantup.com/)
 * [VirtualBox](https://www.virtualbox.org/)
 * [Ansible](http://www.ansible.com/)


Usage
-----

To Deploy the Infrastructure and the app

 * `vagrant up`
 * `ansible-playbook -i test site.yml`


Then you can generate some traffic [here](http://192.168.56.45)

You can consult your traffic on [hq](http://192.168.56.46:9200/_plugin/hq) or play with kibana [here](http://192.168.56.46:5601/app/kibana)


You can find the article [here](article.md)
