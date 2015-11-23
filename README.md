# how-to-clean-up-host-machine

如何清除宿主机

由于目前我们的所有客户的服务器上的服务都切换为 docker 容器运行, 故需要对服务器上的服务、代码进行清理.

如下为相对比较详细的清理步骤

# 清理宿主机


## 清理前的检查


**注意: 需要先检查 lims2 是否被部署, lims2 如果被部署, 有可能引发基础服务删除时候卡死在 MySQL 备份上**
```
dpkg -l | grep lims2
```

如存在 `lims2`, 则需要先进行 prerm 的修改

```
ls -al /var/lib/dpkg/info/lims2*
```

* 针对, lims2 的 prerm 需要对所有内容进行注释（不使用 dpkg 自带的处理机制), 仅返回 `exit 0` 即可
* 针对, lims2.list 文件的内容进行清空(不进行文件 purge)
* 针对, lims2.postrm 文件内容进行清空(不进行 purge 操作处理)

**注意: 需要先检查 gini 是否被部署, gini 如果被部署, 可能引发 nodejs 删除后也会删除 gini 的情况产生**

```
dpkg -l | grep gini
ls -al /usr/local/share/composer/vendor/
gini -v
```

如果发现部署了 `gini`, 则**不要删除 `nodejs`!!!** 

**如果有前台服务, 不应该删除 nodejs、gini 等. 不应该删除 /etc/apt/ 下的 nodejs 的 ppa 源!!!**


**注意: 需要对现有的容器进行检查, 包括基础服务、latest 容器服务**

```
docker ps | grep -E 'mariadb|redis|sphinxsearch|beanstalkd'
```

```
docker ps | grep latest
```

**基础服务必须部署到容器中**, 如果没部署, 先参考 [部署文档](https://github.com/genee-tools/lims2-deploy-doc) 进行部署

## 基础服务删除

**注意: 如下服务均为通用服务, 可通用执行**

```
sudo apt-get remove --purge \
    memcached \
    redis-server \
    mysql-server* \
    clamav* \
    beanstalkd \
    sphinxsearch \
    apache2-utils \
    puppet \
    msmtp \
    nodejs \ #如果有 gini, 不要删!
```

### 修正 memecached 扩展删除后, 导致的 php5-fpm 无法重启的问题

如果 **php5-memcached** 安装, 则有可能会重启所有的 php5-fpm 服务(docker 容器中的 php5-fpm 并不会自动重启, 需要手动重启 **php5-fpm**

```
sudo docker exec -it lims2 supervisorctl restart php5-fpm
```

### 修正删除 MySQL 后 mysql-client 自动被 autoremove 掉的问题

```
sudo apt-get install mysql-client
```

### 删除其他不依赖的包

```
sudo apt-get autoremove --purge
```

**注意!**, `autoremove --purge` 可能会成自动清除不用的 kernel、image, 需要注意 `/initrd.img`、`/vmlinuz` link 文件是否正确

### 删除 cron 中不依赖的 easyrun

```
grep -rn 'easyrun' /etc/cron*
```

easyrun 会在删除 nodejs 的时候进行删除, 如果删除了 nodejs, 则需要手动修改 cron.d 中 easyrun.


### 删除不用的 apt 源、多余文件

```
sudo su 

rm -rf /etc/apt/sources.list.d/apt_genee_in_8800_apt_ppa_launchpad_net_builds_sphinxsearch_rel22_ubuntu.list \
        /etc/apt/sources.list.d/chris-lea-zeromq-precise.list \
        /etc/apt/sources.list.d/ppa_chris_lea_zeromq_precise.list \
        /etc/apt/sources.list.d/builds-sphinxsearch-rel22-precise.list \
        /etc/apt/sources.list.d/chris-lea-zeromq-precise.list.save \
        /etc/apt/sources.list.d/puppetlabs.list.save \
        /etc/apt/sources.list.d/puppetlabs.list \
        /etc/apt/sources.list.d/builds-sphinxsearch-rel22-precise.list.save \
        /etc/apt/sources.list.d/ppa_builds_sphinxsearch_rel22_precise.list \
        /etc/apt/sources.list.d/ppa_builds_sphinxsearch_rel22_precise.list.save \
        /etc/apt/sources.list.d/ppa_chris_lea_zeromq_trusty.list \
        /etc/apt/sources.list.d/chris-lea-redis-server-precise.list \
        /etc/apt/sources.list.d/chris-lea-redis-server-precise.list.save \
        /var/log/sphinxsearch \
        /var/lib/sphinxsearch \
        /etc/init.d/clamav-freshclam \
        /var/lib/clamav \
        /var/log/clamav \
        /var/log/redis \
        /etc/puppet/ \
        /etc/clamav/ \
        /etc/apt/sources.list.d/chris-lea-node_js-precise.list \
        /etc/apt/sources.list.d/chris-lea-node_js-precise.list.save \
        /etc/apt/sources.list.d/deb_nodesource_com_node_0_10.list \
```

**注意, 如果不删除 nodejs, 最后 3 个 list 文件不要进行删除**

**注意, 如果宿主机有前台页面(通常前台页面是在宿主机部署的), 需要注意, 不要删除 php 的 apt 源**

### 删除 puppet 后同步更新 hosts

```
sudo sed -i /puppet.genee.cn/d /etc/hosts
```

### 恢复 msmtprc 

```
sudo touch /etc/msmtprc
sudo chmod 644 /etc/msmtprc
sudo vim /etc/msmtprc
```

内容
```
defaults
tls on

tls_starttls on
tls_certcheck off
syslog LOG_MAIL

account default
host smtp.askep.cn
from askep@askep.cn
auth plain
user askep
password askep
```

## 多余代码删除

```
sudo su

for path in dc-cacs-server node-lims2 icco-server cacs-server gdoor-server glogon-server env-server vidmon-server tszz-server epc-server genee-updater-server; do
    if [ -d "/home/genee/$path" ]; then
        cd "/home/genee/$path" && rm -rf `find . -maxdepth 1 | grep -vE 'config|files|logs|lib'`
    fi
done

```

## 删除 tiandy-nvs 和 haikan-nvs

```
sudo apt-get remove --purge tiandy-nvs haikan-nvs
sudo rm -rf /etc/tiandy-nvs* /etc/haikan-nvs*
sudo apt-get remove --purge libcurl4-openssl-dev:i386 libuuid1:i386
sudo apt-get remove `dpkg --get-selections | grep i386 | awk '{print $1}'`
sudo apt-get autoremove --purge
```

## 宿主机 php 删除

```
sudo apt-get remove --purge php5-*
sudo mkdir /var/lib/php5/
sudo docker exec -it lims2 mkdir /var/lib/php5/
sudo docker exec -it lims2 chown -R www-data:www-data /var/lib/php5/
sudo docker exec -it lims2 supervisorctl restart php5-fpm
sudo rm -rf /etc/apt/sources.list.d/ondrej-php5-oldstable-precise.list
sudo rm -rf /etc/apt/sources.list.d/ondrej-php5-oldstable-precise.list.save
```

## 删除 ganglia 

```
sudo service ganglia-monitor stop 
sudo apt-get remove --purge ganglia-monitor libganglia1
sudo rm -rf /etc/ganglia/
```

## 删除 kill_zombie_puppet.sh 

```
sudo rm -rf /etc/kill_zombie_puppet.sh
```
