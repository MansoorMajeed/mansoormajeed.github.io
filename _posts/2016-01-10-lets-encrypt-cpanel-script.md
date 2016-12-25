---
id: 660
title: '[Script] How to set up Let&#8217;s Encrypt in cPanel/WHM ( Centos 6.x / 7.x )'
date: 2016-01-10T23:04:28+00:00
author: Mansoor
layout: post
guid: https://digitz.org/blog/?p=660
permalink: /lets-encrypt-cpanel-script/
dsq_thread_id:
  - 4479142390
categories:
  - CentOs
  - CenTos 7
  - cPanel
  - Scripts
  - Sysadmin
tags:
  - CentOs
  - CenTos 7
  - cpanel
  - letsencrypt
  - Linux
---
Let&#8217;s be quick and clear. If you&#8217;re here, you don&#8217;t need a preface for Let&#8217;s Encrypt. You probably know how awesome it is. So today I&#8217;ll show you guys how to quickly and easily setup let&#8217;s encrypt in your cPanel server, and install SSL certificates for your domains with ease. Please note that you need a dedicated server/VPS for this. Shared hosting is not supported. So, let&#8217;s get started

&nbsp;

### Setting up Let&#8217;s Encrypt

#### For Centos 6.x

The thing about CentOs 6.x is it comes with Python 2.6 where as Let&#8217;s Encrypt supports Python 2.7+ only. But, installing Python 2.7 in Centos 6.x is pretty simple.

##### Installing Python 2.7 in cPanel ( Centos 6.x )

This step is only for Centos 6.x only

<pre class="toolbar:2 lang:default decode:true"># Install Epel Repository
yum install epel-release

# Install IUS Repository
rpm -ivh https://rhel6.iuscommunity.org/ius-release.rpm

# Install Python 2.7 and Git
yum --enablerepo=ius install git python27 python27-devel python27-pip python27-setuptools python27-virtualenv -y
</pre>

Make sure that the installation was successful.

<pre class="lang:default decode:true">which python2.7

# The above command should display : /usr/bin/python2.7</pre>

So , now we know that Python 2.7 is installed properly.

#### The following steps are common for CentOs 6.x and 7.x

<pre class="toolbar:2 lang:default decode:true "># Install Git
yum -y install git

cd /root

# Clone the letsencrypt Repo

git clone https://github.com/letsencrypt/letsencrypt
cd letsencrypt

# Run the letsencrypt script
./letsencrypt-auto --verbose</pre>

This will take  a few minutes. At the end, you will probably receive a message like _&#8220;No installers are available on your OS yet&#8221;_

That&#8217;s it, you&#8217;re ready to retrieve SSL certificates for your domains. Issue the following command to retrieve an SSL certificate for your domain &#8220;example.com&#8221;

> Note: I have made a simple php script to automate the installation and retrieval of SSL certificates, feel free to skip to that part

<pre class="toolbar:2 lang:default decode:true ">./letsencrypt-auto --text --agree-tos --email you@example.com certonly --renew-by-default --webroot --webroot-path /home/username/public_html/ -d example.com -d www.example.com</pre>

It&#8217;s pretty simple, isn&#8217;t? Now, you will receive something like the following

> Congratulations! Your certificate and chain have been saved at
  
> /etc/letsencrypt/live/example.com/fullchain.pem. Your cert will
  
> expire on 2016-04-08. To obtain a new version of the certificate in
  
> the future, simply run Let&#8217;s Encrypt again.

You can find your certificate and key under /etc/letsencrypt/live/example.com/.

Now you have to install the certificates from the cPanel/WHM interface.

#### Automating SSL certificate installation in cPanel

So I have made a simple PHP script which will do the job for you. Copy the below and save it as &#8220;letsencrypt-cpanel.php&#8221; Simply run the script in the server itself ( php letsencrypt-cpanel.php ), enter the domain name, email address and username for the domain and it will install the SSL certificate for you. You don&#8217;t have to manually install the certificates.

Before running the below script, make sure that you have the cPanel access hash. Issue the command

<pre class="">cat /root/.accesshash
</pre>

It should display a random string. If it does not, login as root in WHM, and search for &#8220;Remote Access&#8221;, and click on it. Now it should generate the access hash. You can verify it by running the above command. Once you have made sure that the accesshash is generated, you can use the script.

{% highlight php %}
<?php
# Please note that no proper validation is done in the script as I'm too lazy for that
# make sure that the domain is pointed to the server's ip correctly
# and, do it at your own risk
# Location of the letsencrypt script
$le = "/root/letsencrypt/letsencrypt-auto";
$handle = fopen("php://stdin","r");
echo "Welcome to Letsencrypt SSL Setup Script\n";
echo "Please Enter the details requested\n";
echo "Domain : ";
$domain = trim(fgets($handle));
echo "cPanel username : ";
$username = trim(fgets($handle));
echo "Email : ";
$email = trim(fgets($handle));

echo "Retrieving the SSL certificates for the domain $domain..!!\n";
$cmd = "$le --text --agree-tos --email $email certonly --renew-by-default --webroot --webroot-path /home/$username/public_html/ -d $domain";
echo "The command is: $cmd";
echo "\n\nAre you sure you wanna continue? If not, press Ctrl+C now\n";
fgets($handle);
$result = shell_exec($cmd);
echo "Command completed: \n$result\n";
echo "Setting up certificates for the domain\n";

$whmusername = 'root';
$hash = file_get_contents('/root/.accesshash');
$query = "https://127.0.0.1:2087/json-api/listaccts?api.version=1&search=$username&searchtype=user";

$curl = curl_init();
curl_setopt($curl, CURLOPT_SSL_VERIFYHOST,0);
curl_setopt($curl, CURLOPT_SSL_VERIFYPEER,0);
curl_setopt($curl, CURLOPT_RETURNTRANSFER,1);
  
$header[0] = "Authorization: WHM $whmusername:" . preg_replace("'(\r|\n)'","",$hash);
curl_setopt($curl,CURLOPT_HTTPHEADER,$header);
curl_setopt($curl, CURLOPT_URL, $query);
$ip = curl_exec($curl);
if ($ip == false) {
        echo "Curl error: " . curl_error($curl);
}
$ip = json_decode($ip, true);
$ip = $ip['data']['acct']['0']['ip'];

$cert = urlencode(file_get_contents("/etc/letsencrypt/live/" . $domain . "/cert.pem"));
$key = urlencode(file_get_contents("/etc/letsencrypt/live/" . $domain . "/privkey.pem"));
$chain = urlencode(file_get_contents("/etc/letsencrypt/live/" . $domain . "/chain.pem"));
$query = "https://127.0.0.1:2087/json-api/installssl?api.version=1&domain=$domain&crt=$cert&key=$key&cab=$chain&ip=$ip";
curl_setopt($curl, CURLOPT_URL, $query);
$result = curl_exec($curl);
if ($result == false) {
        echo "Curl error: " . curl_error($curl);
}
curl_close($curl);
  
print $result;
echo "All Done\n";</pre>
{% endhighlight %}

Please note that I have not added any error catching, so if there is any error in the submitted data, you will probably get some weird error. I&#8217;ll try to add proper error handling in the future. You can find the latest version of the script <a href="https://github.com/MansoorMajeed/Letsencrypt-Cpanel-Installer" target="_blank">Here</a>

And, use it at your own risk 😉 If you need any help, leave a comment.

<div class="synved-social-container synved-social-container-share" style="text-align: center">
  <a class="synved-social-button synved-social-button-share synved-social-size-48 synved-social-resolution-single synved-social-provider-facebook nolightbox" data-provider="facebook" target="_blank" rel="nofollow" title="Share on Facebook" href="http://www.facebook.com/sharer.php?u=https%3A%2F%2Fdigitz.org%2Fblog%2Fwp-admin%2Fexport.php%3Ftype%3Djekyll&#038;t=%5BScript%5D%20How%20to%20set%20up%20Let%E2%80%99s%20Encrypt%20in%20cPanel%2FWHM%20%28%20Centos%206.x%20%2F%207.x%20%29&#038;s=100&#038;p&#091;url&#093;=https%3A%2F%2Fdigitz.org%2Fblog%2Fwp-admin%2Fexport.php%3Ftype%3Djekyll&#038;p&#091;images&#093;&#091;0&#093;=&#038;p&#091;title&#093;=%5BScript%5D%20How%20to%20set%20up%20Let%E2%80%99s%20Encrypt%20in%20cPanel%2FWHM%20%28%20Centos%206.x%20%2F%207.x%20%29" style="font-size: 0px; width:48px;height:48px;margin:0;margin-bottom:5px;margin-right:5px;"><img alt="Facebook" title="Share on Facebook" class="synved-share-image synved-social-image synved-social-image-share" style="display: inline; width:48px;height:48px; margin: 0; padding: 0; border: none; box-shadow: none;" src="https://i0.wp.com/digitz.org/blog/wp-content/plugins/social-media-feather/synved-social/image/social/regular/96x96/facebook.png?resize=48%2C48&#038;ssl=1" data-recalc-dims="1" /></a><a class="synved-social-button synved-social-button-share synved-social-size-48 synved-social-resolution-single synved-social-provider-twitter nolightbox" data-provider="twitter" target="_blank" rel="nofollow" title="Share on Twitter" href="http://twitter.com/share?url=https%3A%2F%2Fdigitz.org%2Fblog%2Fwp-admin%2Fexport.php%3Ftype%3Djekyll&#038;text=Hey%20check%20this%20out" style="font-size: 0px; width:48px;height:48px;margin:0;margin-bottom:5px;margin-right:5px;"><img alt="twitter" title="Share on Twitter" class="synved-share-image synved-social-image synved-social-image-share" style="display: inline; width:48px;height:48px; margin: 0; padding: 0; border: none; box-shadow: none;" src="https://i1.wp.com/digitz.org/blog/wp-content/plugins/social-media-feather/synved-social/image/social/regular/96x96/twitter.png?resize=48%2C48&#038;ssl=1" data-recalc-dims="1" /></a><a class="synved-social-button synved-social-button-share synved-social-size-48 synved-social-resolution-single synved-social-provider-google_plus nolightbox" data-provider="google_plus" target="_blank" rel="nofollow" title="Share on Google+" href="https://plus.google.com/share?url=https%3A%2F%2Fdigitz.org%2Fblog%2Fwp-admin%2Fexport.php%3Ftype%3Djekyll" style="font-size: 0px; width:48px;height:48px;margin:0;margin-bottom:5px;margin-right:5px;"><img alt="google_plus" title="Share on Google+" class="synved-share-image synved-social-image synved-social-image-share" style="display: inline; width:48px;height:48px; margin: 0; padding: 0; border: none; box-shadow: none;" src="https://i1.wp.com/digitz.org/blog/wp-content/plugins/social-media-feather/synved-social/image/social/regular/96x96/google_plus.png?resize=48%2C48&#038;ssl=1" data-recalc-dims="1" /></a><a class="synved-social-button synved-social-button-share synved-social-size-48 synved-social-resolution-single synved-social-provider-reddit nolightbox" data-provider="reddit" target="_blank" rel="nofollow" title="Share on Reddit" href="http://www.reddit.com/submit?url=https%3A%2F%2Fdigitz.org%2Fblog%2Fwp-admin%2Fexport.php%3Ftype%3Djekyll&#038;title=%5BScript%5D%20How%20to%20set%20up%20Let%E2%80%99s%20Encrypt%20in%20cPanel%2FWHM%20%28%20Centos%206.x%20%2F%207.x%20%29" style="font-size: 0px; width:48px;height:48px;margin:0;margin-bottom:5px;margin-right:5px;"><img alt="reddit" title="Share on Reddit" class="synved-share-image synved-social-image synved-social-image-share" style="display: inline; width:48px;height:48px; margin: 0; padding: 0; border: none; box-shadow: none;" src="https://i2.wp.com/digitz.org/blog/wp-content/plugins/social-media-feather/synved-social/image/social/regular/96x96/reddit.png?resize=48%2C48&#038;ssl=1" data-recalc-dims="1" /></a><a class="synved-social-button synved-social-button-share synved-social-size-48 synved-social-resolution-single synved-social-provider-pinterest nolightbox" data-provider="pinterest" target="_blank" rel="nofollow" title="Pin it with Pinterest" href="http://pinterest.com/pin/create/button/?url=https%3A%2F%2Fdigitz.org%2Fblog%2Fwp-admin%2Fexport.php%3Ftype%3Djekyll&#038;media=&#038;description=%5BScript%5D%20How%20to%20set%20up%20Let%E2%80%99s%20Encrypt%20in%20cPanel%2FWHM%20%28%20Centos%206.x%20%2F%207.x%20%29" style="font-size: 0px; width:48px;height:48px;margin:0;margin-bottom:5px;margin-right:5px;"><img alt="pinterest" title="Pin it with Pinterest" class="synved-share-image synved-social-image synved-social-image-share" style="display: inline; width:48px;height:48px; margin: 0; padding: 0; border: none; box-shadow: none;" src="https://i2.wp.com/digitz.org/blog/wp-content/plugins/social-media-feather/synved-social/image/social/regular/96x96/pinterest.png?resize=48%2C48&#038;ssl=1" data-recalc-dims="1" /></a><a class="synved-social-button synved-social-button-share synved-social-size-48 synved-social-resolution-single synved-social-provider-linkedin nolightbox" data-provider="linkedin" target="_blank" rel="nofollow" title="Share on Linkedin" href="http://www.linkedin.com/shareArticle?mini=true&#038;url=https%3A%2F%2Fdigitz.org%2Fblog%2Fwp-admin%2Fexport.php%3Ftype%3Djekyll&#038;title=%5BScript%5D%20How%20to%20set%20up%20Let%E2%80%99s%20Encrypt%20in%20cPanel%2FWHM%20%28%20Centos%206.x%20%2F%207.x%20%29" style="font-size: 0px; width:48px;height:48px;margin:0;margin-bottom:5px;margin-right:5px;"><img alt="linkedin" title="Share on Linkedin" class="synved-share-image synved-social-image synved-social-image-share" style="display: inline; width:48px;height:48px; margin: 0; padding: 0; border: none; box-shadow: none;" src="https://i1.wp.com/digitz.org/blog/wp-content/plugins/social-media-feather/synved-social/image/social/regular/96x96/linkedin.png?resize=48%2C48&#038;ssl=1" data-recalc-dims="1" /></a><a class="synved-social-button synved-social-button-share synved-social-size-48 synved-social-resolution-single synved-social-provider-mail nolightbox" data-provider="mail" rel="nofollow" title="Share by email" href="mailto:?subject=%5BScript%5D%20How%20to%20set%20up%20Let%E2%80%99s%20Encrypt%20in%20cPanel%2FWHM%20%28%20Centos%206.x%20%2F%207.x%20%29&#038;body=Hey%20check%20this%20out:%20https%3A%2F%2Fdigitz.org%2Fblog%2Fwp-admin%2Fexport.php%3Ftype%3Djekyll" style="font-size: 0px; width:48px;height:48px;margin:0;margin-bottom:5px;"><img alt="mail" title="Share by email" class="synved-share-image synved-social-image synved-social-image-share" style="display: inline; width:48px;height:48px; margin: 0; padding: 0; border: none; box-shadow: none;" src="https://i1.wp.com/digitz.org/blog/wp-content/plugins/social-media-feather/synved-social/image/social/regular/96x96/mail.png?resize=48%2C48&#038;ssl=1" data-recalc-dims="1" /></a>
</div>