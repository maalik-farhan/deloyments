# Jenkins SetUp

- Install the default-jdk

```
apt-get install default-jdk
```

- verify that java

```
java --version
```

- Add the jenkins repository
```
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```

- Update the package index
```
sudo apt-get update
```

- Install jenkins
```
sudo apt-get jenkins
```

- Start the Jenkins service
```
systemctl start jenkins
```
- Enable jenkins service
```
systemctl enable jenkins
```

- To get your administrator password
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
