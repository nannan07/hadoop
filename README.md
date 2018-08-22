
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>hadoop (一)Cloudera Manager一键安装</title>
<link rel="stylesheet" href="https://csdn.net/res-min/themes/base.css" />
<script type="text/javascript" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
<script type="text/x-mathjax-config">
MathJax.Hub.Config({ TeX: { equationNumbers: {autoNumber: "all"},extensions: ["AMSmath.js","AMSsymbols.js","noErrors.js","noUndefined.js"] } });
</script>
</head>
<body><div class="container"><h2 id="hadoop-一cloudera-manager一键安装">hadoop (一)Cloudera Manager一键安装</h2>



<h4 id="一linux环境准备">一、Linux环境准备</h4>

<p>cm安装要求，具体配置参考<a href="http://www.cloudera.com/downloads/manager/5-6-0.html">cloudera官网</a></p>

<ol>
<li>centos需要使用6.4以上而且是64位的，同时对内存以及磁盘的空间都有严格的要求。 </li>
<li>只有一台服务器的情况下 ，使用虚拟机安装了三台linux系统，如果一台服务器而且内存不够大的话，基本无法运行</li>
</ol>



<h4 id="二准备工作">二、准备工作</h4>

<p>首先，配置给三台服务器配置好ip：<code>vim /etc/sysconfig/network-scripts/ifcfg-eth0</code></p>

<p>设置如下：</p>

<pre><code>    DEVICE="eth0"
    BOOTPROTO="static"
    HWADDR="00:0C:29:4E:BB:D5"
    NM_CONTROLLED="yes"
    ONBOOT="yes" 
    TYPE="Ethernet"
    UUID="b530dd2b-639e-4746-ba0d-254c78811de1"
    IPADDR="172.31.26.200"
    NETMASK="255.255.255.0"
    GATEWAY="172.31.26.254"
    DNS1="****"（这个自己设置一下）
</code></pre>

<p>然后，执行service network restart重启一下网络。  <br>
接下来，就是关闭SELinux、关闭防火墙。  <br>
关闭SELinux：  <br>
暂时关闭：setenforce 0 搭环境一定要注意关闭，像搭httpd服务的时候必须关掉。 <br>
执行<code>vi /etc/selinux/config</code>， 设置如下内容便可永久关闭</p>

<pre><code>    SELINUX=disabled 
</code></pre>

<p>关闭防火墙：  <br>
1) 重启后生效  <br>
开启： chkconfig iptables on  <br>
关闭： chkconfig iptables off</p>

<p>2) 即时生效，重启后失效  <br>
开启： service iptables start  <br>
关闭： service iptables stop </p>

<p>接下来，就是修改一下hosts文件，像大数据中一般使用主机名。  <br>
执行：<code>vim /etc/hosts</code>，添加</p>

<pre><code>    172.31.26.200 bigData200
    172.31.26.222 bigData222
    172.31.26.223 bigData223
</code></pre>

<p>三个节点都要添加。  <br>
如果某个过程中遇到错误，大家可以自行百度，都可以找到相应的解决办法。</p>



<h4 id="三ssh互访">三、ssh互访</h4>

<p>建立Master到每一台Slave的SSH受信证书。由于Master将会通过SSH启动所有Slave的Hadoop，所以需要建立单向或者双向证书保证命令执行时不需要再输入密码。在Master和所有的Slave机器上执行：<code>ssh-keygen -t rsa</code>。执行此命令的时候，看到提示只需要回车。然后就会在/root/.ssh/下面产生id_rsa.pub的证书文件，通过scp将Master机器上的这个文件拷贝到Slave上（记得修改名称），例如：<code>scp root@masterIP:/root/.ssh/id_rsa.pub /root/.ssh/46_rsa.pub</code>，然后执行<code>cat /root/.ssh/46_rsa.pub &gt;&gt;/root/.ssh/authorized_keys</code>，建立authorized_keys文件即可，可以打开这个文件看看，也就是rsa的公钥作为key，user@IP作为value。从slave到master反向也是一样的操作。</p>

<p>然后每台服务器上都修改ssh的配置文件：<code>vim /etc/ssh/sshd_config</code></p>

<pre><code>     RSAAuthentication yes
     PubkeyAuthentication yes
     AuthorizedKeysFile  .ssh/authorized_keys
</code></pre>

<p>执行：<code>vim /etc/ssh/ssh_config</code>：</p>

<pre><code>     StrictHostKeyChecking no
</code></pre>

<p>最后，可以看下主节点和子节点上的authorized_keys  <br>
执行<code>cat /root/.ssh/authorized_keys</code> <br>
也就是说，你要无密码登录哪台主机，就得在那台主机上有一个授权的key。</p>



<h4 id="四ntp时间同步配置">四、ntp时间同步配置</h4>

<p>这个配置，其实是为了保证slave节点与主节点的时间保持一致。 如果没有安装ntp服务，则先安装：  <br>
ntp安装：  <br>
<code>yum -y install ntp</code> /yum安装NTP服务/  <br>
<code>chkconfig –add ntpd</code> /添加NTP/  <br>
<code>chkconfig ntpd on</code> /开机自启动NTP/</p>

<p>设置：</p>

<p>1.在iDriller主节点上编辑配置文件/etc/ntp.conf  <br>
<code>vim /etc/ntp.conf</code> <br>
去掉一下两行前面的#号</p>

<pre><code>server 127.127.1.0     # local clock
fudge  127.127.1.0 stratum 10
</code></pre>

<p>2. 在iDriller子节点上分别编辑配置文件/etc/ntp.conf  <br>
<code>vim /etc/ntp.conf</code> <br>
在#restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap行下面增加行： </p>

<pre><code>    server bigData200
</code></pre>

<p>启动ntp服务： </p>

<pre><code>service ntpd start
</code></pre>

<p>验证： </p>

<pre><code>watch ntpq -p
</code></pre>



<h4 id="五主节点上安装clush">五、主节点上安装clush</h4>

<p>安装clush需要先配置好ssh免登陆。Clush可以在集群上并行执行shell命令并收集命令输出，所以分发文件就比较方便 <br>
先去下载clustershell-1.6-1.el6.noarch.rpm，然后安装 <br>
<code>rpm -ivh clustershell-1.6-1.el6.noarch.rpm</code> <br>
这个只需要安装在主节点上，安装好后，执行  <br>
<code>vim /etc/clustershell/groups</code>，然后清空所有设置：</p>

<pre><code>all:bigData200,bigData222,bigData223
slaves:bigData222,bigData223
</code></pre>

<p>然后，我们可以创建一个文件   <code>touch /tmp/testfile</code>，进行测试  <br>
<code>clush -g all –copy /tmp/testfile –dest /tmp/</code>，看是否分发到从节点上。</p>



<h4 id="六yum本地源配置">六、yum本地源配置</h4>

<p>这一步可以说是最麻烦的，麻烦就在准备源，也就是rmp包，因为安装某个包时会有先安装很多依赖，所以这些包都得准备好。另外，为什么要使用本地源呢，因为大数据服务器一般是不会联网的，同时不使用本地源传输速度非常慢。  <br>
首先，需要安装一个httpd服务，这个可以不安装在集群的节点上，都行。</p>

<pre><code>yum -y install httpd
service httpd restart
chkconfig httpd on
</code></pre>

<p>然后，http默认是将文件全部放在/var/www/html/下的： <br>
<img src="https://img-blog.csdn.net/20180822141742795?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDc4MzExMg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="这里写图片描述" title=""></p>

<p>这里，把cm相关的包放在cm目录下，把相关的常用包放在iso下，把升级包放在parcels下，然后启动服务后直接在浏览器中便可访问，如下 <br>
<img src="https://img-blog.csdn.net/20180822141840589?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDc4MzExMg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="<code>这里写图片描述</code>" title=""></p>



<h4 id="七分发yum配置文件">七、分发yum配置文件</h4>

<p>这里，我们先执行：<code>cd /etc/yum.repos.d/</code> <br>
然后创建一个目录bak，把这些默认源移到bak下，添加本地源cm.repo和iso.repo: </p>

<p><strong>cm.repo:</strong></p>



<pre class="prettyprint"><code class=" hljs ini"><span class="hljs-title">[cm]</span>
<span class="hljs-setting">name=<span class="hljs-value">cm</span></span>
<span class="hljs-setting">baseurl=<span class="hljs-value">http://rzx161/cm</span></span>
<span class="hljs-setting">enabled=<span class="hljs-value"><span class="hljs-number">1</span></span></span>
<span class="hljs-setting">gpgcheck=<span class="hljs-value"><span class="hljs-number">0</span></span></span></code></pre>

<p><strong>iso.repo:</strong></p>



<pre class="prettyprint"><code class=" hljs ini"><span class="hljs-title">[iso]</span>
<span class="hljs-setting">name=<span class="hljs-value">iso</span></span>
<span class="hljs-setting">baseurl=<span class="hljs-value">http://rzx161/iso</span></span>
<span class="hljs-setting">enabled=<span class="hljs-value"><span class="hljs-number">1</span></span></span>
<span class="hljs-setting">gpgckeck=<span class="hljs-value"><span class="hljs-number">0</span></span></span>
<span class="hljs-setting">gpgkey=<span class="hljs-value">http://<span class="hljs-number">172.31</span>.<span class="hljs-number">25.161</span>/iso/RPM-GPG-KEY-CentOS-<span class="hljs-number">6</span></span></span></code></pre>

<p>然后使用clush分发到另外两条从节点上去，执行 <br>
<code>clush -g all --copy *.repo --dest /etc/yum.repos.d/</code></p>



<h4 id="八所有节点安装mysql-jdbc驱动">八、所有节点安装MySQL JDBC驱动</h4>

<p>先卸载原有mysql  <br>
<code>rpm -qa | grep -i mysql | xargs rpm -e –nodeps</code> <br>
然后安装驱动  <br>
<code>yum -y install mysql-connector-java</code></p>



<h4 id="九在主节点上安装cm-server和mysql-server">九、在主节点上安装cm server和mysql server</h4>

<p>执行  <br>
<code>yum -y install cloudera-manager-server*</code>  <br>
<code>yum -y install mysql-server</code> <br>
分别安装cm server和mysql。  <br>
设置服务为开机启动：  <br>
chkconfig cloudera-scm-server on  <br>
chkconfig cloudera-scm-server-db on  <br>
chkconfig mysqld on  <br>
启动mysql：service mysqld start  <br>
在mysql中创建对应用户和数据库：</p>



<pre class="prettyprint"><code class=" hljs oxygene">mysql&gt;<span class="hljs-keyword">create</span> database cmf <span class="hljs-keyword">default</span> character <span class="hljs-keyword">set</span> utf8 collate utf8_general_ci;
mysql&gt;grant all <span class="hljs-keyword">on</span> cmf.* <span class="hljs-keyword">to</span> <span class="hljs-string">'cmf'</span>@<span class="hljs-string">'localhost'</span> identified <span class="hljs-keyword">by</span> <span class="hljs-string">'cmf'</span>;
mysql&gt;flush privileges;
mysql&gt;<span class="hljs-keyword">exit</span>;</code></pre>

<p>编辑/etc/cloudera-scm-server/db.properties，将cmf库正确配置：</p>



<pre class="prettyprint"><code class=" hljs avrasm"><span class="hljs-keyword">com</span><span class="hljs-preprocessor">.cloudera</span><span class="hljs-preprocessor">.cmf</span><span class="hljs-preprocessor">.db</span><span class="hljs-preprocessor">.type</span>=mysql
<span class="hljs-keyword">com</span><span class="hljs-preprocessor">.cloudera</span><span class="hljs-preprocessor">.cmf</span><span class="hljs-preprocessor">.db</span><span class="hljs-preprocessor">.host</span>=localhost
<span class="hljs-keyword">com</span><span class="hljs-preprocessor">.cloudera</span><span class="hljs-preprocessor">.cmf</span><span class="hljs-preprocessor">.db</span><span class="hljs-preprocessor">.name</span>=cmf
<span class="hljs-keyword">com</span><span class="hljs-preprocessor">.cloudera</span><span class="hljs-preprocessor">.cmf</span><span class="hljs-preprocessor">.db</span><span class="hljs-preprocessor">.user</span>=cmf
<span class="hljs-keyword">com</span><span class="hljs-preprocessor">.cloudera</span><span class="hljs-preprocessor">.cmf</span><span class="hljs-preprocessor">.db</span><span class="hljs-preprocessor">.password</span>=cmf</code></pre>

<p>启动cm：  <br>
service cloudera-scm-server-db start  <br>
service cloudera-scm-server start</p>



<h4 id="十访问httpip或者配置了的hostname7180-分发安装包">十、访问<a href="http://ip">http://ip</a>（或者配置了的hostname）:7180/ 分发安装包</h4>

<p>在浏览器访问这个地址便可看到 <br>
密码默认是admin。  <br>
登陆进去后就是按照步骤操作</p>

<p>这个位置容易失败，主要是系统版本的原因，导致一些依赖有问题，真是悲剧。这个过程中要填一些源，就把刚才搭建httpd中的路径<a href="http://IP/cm/">http://IP/cm/</a>和<a href="http://IP/parcels/">http://IP/parcels/</a>填写进去即可。</p>



<h4 id="十一添加安装服务并调整相应的配置">十一、添加安装服务并调整相应的配置</h4>

<p><img src="https://img-blog.csdn.net/20180822142634998?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDc4MzExMg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="这里写图片描述" title=""> <br>
cm中可以添加很多服务，我们可以选择需要的服务先安装，之后也可以新增服务。  <br>
最好服务都成功后，并按照提示修改下相应的配置，便可看到： <br>
<img src="https://img-blog.csdn.net/20180822142722464?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MDc4MzExMg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="这里写图片描述" title=""> <br>
在cm可以修改对于服务的配置文件，还可以执行命令，还可以监控服务等</p>

<p>⚠️服务器不同，配置环境不同，过程中会有相应的问题，如果过程中有维提岛的问题，自行百度解决</p></div></body>
</html>
