# ansible-disa-stigs
Generic method to secure servers with Ansible and the disa-stig module

In the lab, I set the host-only IP address to 172.16.0.10

## Preparatory Work
Workstation/Bastion_Host

*assuming you are on RHEL7 and subscribed to RHN*
Install PIP the Red Hat way:
```sh
subscription-manager repos --enable rhel-server-rhscl-7-rpms
yum install python27-python-pip
```

Enable ``pip`` in your shell:
```sh
scl enable python27 bash
which pip
pip -V
```

Install ansible and passlib, required for disa-stig module.
```sh
sudo -H pip install ansible==2.4.1
sudo -H pip install passlib==1.7.1
```

For performance gains with Ansible, install mitogen.
```sh
sudo -H pip install mitogen
```

Push my SSH key to the instance as both my  user account and also the root account.
```sh
ssh-copy-id -i ~/.ssh/id_rsa goldenleg@172.16.0.10
```

My test server has the user ``goldenleg`` setup as an admin user (in the wheel group)
Make sure that the file ``/etc/sudoers`` has this block:
```sh
## Same thing without a password
# %wheel	ALL=(ALL)	NOPASSWD: ALL
goldenleg ALL=(ALL) NOPASSWD: ALL 
```

## Pre-install openscap in order to captue pre-STIG lockdown status
*You really don't need to do this unless you want to know what changed*

SSH to test instance, elevate to root
```sh
yum install -y openscap openscap-utils scap-security-guide wget
mkdir /root/Compliance
chmod 0700 /root/Compliance
cd /root/Compliance

wget http://www.redhat.com/security/data/oval/com.redhat.rhsa-all.xml
wget http://www.redhat.com/security/data/metrics/com.redhat.rhsa-all.xccdf.xml

oscap xccdf eval --results /var/tmp/$(hostname).pre.patch.comp.results.xml --report /var/tmp/$(hostname).pre.patch.compliance.results.html com.redhat.rhsa-all.xccdf.xml

oscap xccdf eval --profile stig-rhel7-disa --results /var/tmp/$(hostname).pre.compliance.results.xml --report /var/tmp/$(hostname).pre.compliance.report.html --cpe /usr/share/xml/scap/ssg/content/ssg-rhel7-cpe-dictionary.xml /usr/share/xml/scap/ssg/content/ssg-rhel7-xccdf.xml
```

# Pre-install for AIDE (Advanced Intrusion Detection Engine)
Install software and run first time to initialize the database:
```sh
yum install -y aide
cp /etc/aide.conf /etc/aide.conf_orig

sed -i '/^NORMAL[ ]=.*$/a ## Custom names\nSizeOnly = s+b\nSizeAndChecksum = s+b+md5+sha1\nReallyParanoid = p+i+n+u+g+s+b+m+a+c+md5+sha1+rmd160+tiger\n\n' /etc/aide.conf

sed -i '/Put file matches before directories/a /boot   ReallyParanoid\n/bin    ReallyParanoid\n/sbin   ReallyParanoid\n' /etc/aide.conf
sed -i '/sbin   ReallyParanoid/a /lib    ReallyParanoid\n/lib64  ReallyParanoid\n/opt    ReallyParanoid\n' /etc/aide.conf
sed -i '/opt    ReallyParanoid/a /usr    ReallyParanoid\n/root   ReallyParanoid\n' /etc/aide.conf
printf '!/opt/rh/python27/root/.*\n' >> /etc/aide.conf

aide  --init
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.pre.stig.gz

```


## Playbook install and run:
Get the repo
```sh
mkdir -p ~/SourceCode/Education/
cd ~/SourceCode/Education
git clone git@github.com:DevOpsMaestro/ansible-disa-stigs.git
cd ansible-disa-stigs
```

Manually adjust the ``hosts`` file inventory to the IP address of the instance under ``[servers]``
Then run:
```sh
ansible-playbook -vvv ./rhel7-stig-playbook.yml
```


## Post-Check STIG compliance after STIG'ing with Ansible
*You really don't need to do this unless you want to know what changed*
ssh to test instance and elevate your user to root
```sh
cd /root/Compliance
oscap xccdf eval --results /var/tmp/$(hostname).post.patch.comp.results.xml --report /var/tmp/$(hostname).post.patch.compliance.results.html com.redhat.rhsa-all.xccdf.xml

oscap xccdf eval --profile stig-rhel7-disa --results /var/tmp/$(hostname).post.compliance.results.xml --report /var/tmp/$(hostname).post.compliance.report.html --cpe /usr/share/xml/scap/ssg/content/ssg-rhel7-cpe-dictionary.xml /usr/share/xml/scap/ssg/content/ssg-rhel7-xccdf.xml
```
 
*On Laptop*
```sh
mkdir ~/OpenSCAP_Results
cd ~/OpenSCAP_Results/
scp user_name@172.16.0.10:/var/tmp/* .
```

Review the output of both ``$(hostname).pre.compliance.report.html`` and ``$(hostname).post.compliance.report.html`` files in your browser.

## Post-check AIDE for what the STIG did
*SSH back into the instance/server*
```sh
unalias cp
cp -f /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.from.inside.ansible.stig.run.gz
cp -f /var/lib/aide/aide.db.pre.stig.gz /var/lib/aide/aide.db.gz
aide --check | tee -a /var/tmp/aide_report.txt
```
