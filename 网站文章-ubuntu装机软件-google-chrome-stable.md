#### Ubuntu安装Chrome稳定版(google-chrome-stable)

* 推荐PPA方法，免翻墙

1. `wget -q -O - https://raw.githubusercontent.com/longhr/ubuntu1604hub/master/linux_signing_key.pub | sudo apt-key add`

2. `sudo sh -c 'echo "deb [ arch=amd64 ] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google-chrome.list`

3. `sudo apt-get update`

4. `sudo apt-get install google-chrome-stable`

* 安装Google Chrome unstable 版本：
`sudo apt-get install google-chrome-beta`
* 安装Google Chrome beta 版本：
`sudo apt-get install google-chrome-unstable`

