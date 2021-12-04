S36 Teknik Kurulum Mülakatı
===========================

#### Sanal Makinenin Hazırlanması
Öncelikle sanal makine çalıştırabileceğim bir hipervizör indirdim. Gpl lisansı altında yayınlanmış, özgür bir yazılım olan [VirtualBox](https://www.virtualbox.org/wiki/Downloads) programını tercih ettim.

Ardından sanal makine üzerine kurabilmek için [CentOS 8 iso](http://ftp.itu.edu.tr/Mirror/CentOS/8.4.2105/isos/x86_64/) dosyasını indirdim.

Yeni bir sanal makine oluşturup indirdiğim iso dosyasını mount ettim. Konfigurasyonda 2Gb ram, dinamik olarak genişleyen 8gb  disk seçtim. Network ayarlarını bridge mode a alıp Vm in ip almasını sağladım. Bridge modunda sanal makine ağa bağlanmış diğer bir bilgisayar gibi davranacaktır(Ağ ayarları Nat şeklinde bırakılıp port forwarding ile vm ve host arasında bağlantı da yapılabilir. Örneğin host:8080->vm:80, sonrasında host makineden localhost'a ssh yapılabilir.). Bunu takiben sanal makineyi çalıştırdım.

Centos 8 kurulum ekranında ağ, saat ve tarih, klavye ayarlarını tamamlayıp, parolası karmaşık olan kubison kullanıcısını ekleyip o kullanıcıya admin yetkisi verdim ve minimal installation seçtim.  

Host makine ve sanal makine arasında network problemi olmadığını ping atarak doğruladım. 

#### Ssh Key Based Authentication Yapılması
Wsl(Windows Subsystem for Linux) kullandığım windows makinemde Ubuntu'yu açtım. Key dosyasını oluşturdum. Karmaşık bir passphrase seçtim.

	$ssh-keygen

Parolasız şekilde auth yapılabilmesi için oluşturulan key dosyalarından public key olanı sanal makineye eklenmeli. Public Key dosyasını sanal makinenin authorized key dosyasına kopyaladım.

Secure copy ile kopyalamanın yanında vm ile host arasında paylaşılmış bir klasör oluşturup key dosyasını o şekilde aktarmak da başka bir yöntem olarak kullanılabilir.

	$scp $HOME/.ssh/id_rsa.pub kubison@192.168.1.109:/$HOME/.ssh/authorized_keys

Parolamı girmemin ardından dosya kopyalandı. Sanal makineye ssh ile bağlantı yaptım passphrase girmemin ardından parolasız şekilde ssh yapabilir hale geldim.

Ardından etc/ssh/ssh_config dosyasının editleyip password authenticationı no yaptım, ssh servisini restart edip parola ile ssh bağlantısı yapmayı kapadım.
	
	$sudo vi /etc/ssh/ssh_config


	/etc/ssh/ssh_config
	.
	.
	PasswordAuthentication no
	.
	.

#### Firewall Konfigurasyonun Yapılması
Açık portları görüntülemek için

	$sudo firewall-cmd --list-all

komutunu çalıştırdım. Cockpit ve ssh servislerinin izinli olduğunu gördüm. Cockpit servisinin firewall iznini kapatıp sadece ssh ve web service portlarına izin verdim.

	$sudo firewall-cmd --zone=public --permanent --remove-service cockpit
	$sudo firewall-cmd --zone=public --permanent --add-service http
	$sudo firewall-cmd --reload

Ardından host makinemdeki nmap ile hangi portların açık olduğunu tekrardan kontrol edip istediğim ayarların yapıldığından emin oldum.

#### Apache'nin Kurulması
Statik web sitelerinde Nginx'in hız bakımından -çalışma mantığı sebebiyle- daha hızlı olduğunu biliyorum lakin daha önce çalıştığımda konfigurasyonunu sadece reverse proxy ve load balancing amaçlı kurgulamıştım. Nginx ile virtual domain kurgulaması yapmadığımdan, zaman kaybetmemek adına Apache Web Server'ı seçtim ve kurdum.

	$sudo yum install httpd

Apache'yi çalıştırdım.

	$sudo systemctl start httpd

Makine açılıdığından itibaren servisin çalışması için enable ettim.
	
	$sudo systemctl enable httpd

Host üzerinde tarayıcı ile sanal makine ipsine gidince Apache welcome sayfasını görüp Apache'nin çalıştığından emin oldum.

#### Virtual Hostların Ayağa Kaldırılması
Birden fazla websitesinin bir Apache sunucusunda barındırılması için Virtual Host konfigürasyonu yapılması gerekir.

Her websitesi için ayrı conf dosyası hazırlayıp /etc/httpd/conf.d/ klasörünün içine attım. Conf.d dizinin son hali şu şekilde idi. 

	/etc/httpd/conf.d		
		->zubizu.com.conf
		->s36ozgur.com.conf

Dikkat edilmesi gereken husus ana konfigurasyon dosyası olan /etc/httpd/conf/httpd.conf dosyasında " IncludeOptional conf.d/\*.conf " ayarının yorum satırı şeklinde olmaması. Bu ayar, dizin altındaki konfigurasyonların Apache tarafından okunmasını sağlıyor.

Conf dosyalarının içlerini ise şu şekilde düzenledim.

###### zubizu.com.conf

	<VirtualHost *:80>
    ServerName zubizu.com
    ServerAlias www.zubizu.com #hem zubizu.com hem de www.zubizu.com'dan erişilebilir kılmak için alias'ı böyle verdim
    DocumentRoot /var/www/zubizu.com/public_html

    <Directory /var/www/zubizu.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
    </Directory>

    ErrorLog /var/log/httpd/zubizu.com-error.log
    CustomLog /var/log/httpd/zubizu.com-access.log combined
	</VirtualHost>

###### s36ozgur.com.conf

	<VirtualHost *:80>
    ServerName s36ozgur.com 
    ServerAlias xn--s36zgr-yxa4c.com ##türkçe karakterli olabilmesi için punnycode'a çevirilmesi gerekir
    DocumentRoot /var/www/s36ozgur.com/public_html

    <Directory /var/www/s36ozgur.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
    </Directory>

    ErrorLog /var/log/httpd/s36ozgur.com-error.log
    CustomLog /var/log/httpd/s36ozgur.com-access.log combined
	</VirtualHost>

Her ne kadar DNS kaydı ile ilgilenmenize gerek yok deseniz de test yapabilmek için host makinemin DNS kayıtlarını da şu şekilde güncellemek durumunda kaldım.
	
	192.168.1.109	zubizu.com
	192.168.1.109	www.zubizu.com
	192.168.1.109	s36ozgur.com
	192.168.1.109 	xn--s36zgr-yxa4c.com ##punnycode çevirilmiş hali 

conf dosyalarında belirttiğim dizinleri oluşturdum. Kullanıcı ve grup sahipliklerini apache yaptım. Bu olası bir sızma karşısında Apache kullanıcısının root yetkisi olmadığından bir üst dizine çıkamamasını sağlayacaktır. 

	$sudo mkdir -p /var/www/s36ozgur.com
	$sudo mkdir -p /var/www/zubizu.com
	$sudo chown -R apache:apache /var/www/s36ozgur.com/public_html
	$sudo chown -R apache:apache /var/www/zubizu.com/public_html

Test edebilmek için public_html dizinlerinin altına index.html dosyaları oluşturdum. Virtual Hostların çalıştığından emin oldum.

Bu adımda zorlandığım yer türkçe karakterli Alias ekleme kısmı idi(s36özgür.com). Daha önce yapmadığımdan araştırmam gerekti, internette uzun süre aradıktan sonra punnycode'a çevrilip yazılırsa olabileceğini gördüm.

s36özgür.com için ayrı bir virtual host mu istediniz yoksa alias olarak eklenmesi yeterli oluyormu onu anlayamadım. Bu haliyle de s36özgür.com'a giden istekler aynı Wordpress'e gidecektir.Ben Alias eklenmenin yeterli olduğuna inandığımdan bu şekilde devam edip ayrı bir virtual host conf'u yazmadım. 
Eğer ki Virtual Host oluşturulması istendiyse benzer bir config yazılmalı, sonuna ise redirect eklenmelidir. Böylece gelen istekler oraya yönlendirilebilir. 
Örneğin

	NameVirtualHost *:80
	<VirtualHost *:80>
	   ServerName xn--s36zgr-yxa4c.com
	   DocumentRoot /var/www/s36ozgur1.com/public_html/
	   Redirect  http://s36ozgur.com
	</VirtualHost>


#### Wordpress'in Kurulması

Wordpress kurulum için php-mysql(mariadb)-apache üçlüsüne ihtiyaç duyuyor, apache yüklü ve konfigürasyonu yapılmış olduğundan php ve mariadb ile ilerledim.

	$sudo yum  install php-mysqlnd php-fpm php-json mariadb-server httpd

Wordpressi indirip zipten çıkarabilmek için de tar ve curl kurdum.
	
	$sudo yum install tar curl

Php'nin indiğinden emin oldum.
	
	$php --version

Mariadb veritabanını çalıştırdım. Vm her kapanıp açıldığında servisin otomatik başlaması için enable ettim.

	$sudo systemctl start mariadb
	$sudo systemctl enable mariadb

MariaDB için güvenli kurulumunu yapıp root ile remote bağlanmayı kapadım. Ardından wordpress için bir db oluşturdum. Bu dbyi kullanacak kullanıcının o dbden başka hiç bir dbye erişemeyeceği şekilde yetki verdim.

	$mysql_secure_installation
	mysql> CREATE DATABASE wordpress;
	mysql> CREATE USER `wordpress_user`@`localhost` IDENTIFIED BY 'pass';
	mysql> GRANT ALL ON wordpress.* TO `wordpress_user`@`localhost`;
	mysql> FLUSH PRIVILEGES;
	
Veritabanın ardından wordpress'in son veriyonunu indirdim.
	
	$curl https://wordpress.org/latest.tar.gz --output /tmp/wordpress.tar.gz

Rardan çıkardığım Wordpress'i s36ozgur.com/public_html/ dizinin altına kopyalayıp sahipliğini apache kullanıcısı yaptım ve GUI üzerinden kuruluma devam etmeye çalıştım. GUI hata verip ilerletmeyince wp-config-sample.php dosyasının adını değiştirip wp-config.php yaptım. Db bilgilerini php dosyasının içine girip güncelledim. Sayfaya tekrar istek attığımda wordpress GUIsi karşıma geldi. Kuruluma devam edip kullanıcı adı ve parola belirledim. 

GUI üzerinden new post diyip bir post yazıp dosya olarak host makinesindeki bir resmi ekledim. Postları SEO uyumlu hale getirebilmek için wp admin panelinden Settings-Permalink Settings'den Day and name seçeneğini seçip kaydettim.

Save Changes dememe rağmen kaydetmeyince ben de .htacess dosyasını oluşturup içine permalinkde değiştirdiğim ayarları ekledim. Bu da problemi çözmedi.

İnternetteki araştırmamın ardından problemin Selinux kaynaklı olduğunu anladım. Selinux .htacess dosyasına yazılmasına izin vermiyordu. Selinuxu geçici olarak permissive moda aldım. Wp, .htacess dosyasını güncelleyebildi. Bypass etmek için wordpress klasörünün selinux context'ini değiştirecek bir komut kullanmam gerektiğini gördüm. 
	
	$sudo setenforce permissive

Selinux bu haliyle sadece log üretiyor lakin engellemesi gereken aksiyonları engellemiyor.

	$sudo chcon -t httpd_sys_rw_content_t /var/www/s36ozgur.com/public_html/ -R
Bu komut ile, public html altındaki, Apache tarafından kullanılan klasörlerin, uygulamalar tarafından (Wp)  okuma ve yazma iznine sahip olması sağlanıyor. Selinux buradaki aktiviteleri ona göre değerlendiriyor ve bu komut ile izin verdiğimizden problem yaşanmıyor. 

	$sudo setenforce enforcing

Her şeyin yolunda olduğunu test ettikten sonra selinux'u tekrardan enforcing moda aldım.

Dolayısıyla Wordpress üzerinde SEO uyumlu postlar oluşturup içine dosya yükleyebildim.

#### Zubizu.com Sayfalarının Hazırlanması

Halihazırda sunucuda php yüklü olduğundan istenilen web sitesini php ile yazdım. 100 kere "Kullanıcılarımın kişisel verilerini toplamayacağım." yazdırabilmek için for loopu kullandım.

```php
	<!DOCTYPE html>
	<html>
	<body>
	<title>Zubizu.com</title>
	<h1>Zubizu.com.</h1>
	<?php
	for ($x = 0; $x < 100; $x++) {
	  echo "Kullanıcılarımın kişisel verilerini toplamayacağım.<br>";
	}
	?>
	</body>
	</html>
```
index.html olan dosya adını index.php olarak güncelledim ve çalışır hale getirdim.

yonetim isimli bir klasörün içine index.php dosyası oluşturdum. 

```php
<?php
   ob_start();
   session_start();
?>

<?
   // error_reporting(E_ALL);
   // ini_set("display_errors", 1);
?>

<html lang = "en">

   <head>
      <title>zubizu.com Admin Paneli</title>
   </head>

   <body>

      <h2>Kullanici Adi ve Parola</h2>
      <div class = "container form-signin">

         <?php
            $msg = '';

            if (isset($_POST['login']) && !empty($_POST['username'])
               && !empty($_POST['password'])) {

               if ($_POST['username'] == 'ad.soyad' &&
                  $_POST['password'] == 'parola') {
                  echo 'Dogru kullanici adi ve parola girdiniz. ';
               }else {
                  $msg = 'Yanlis kullanici adi veya parola girdiniz.';
               }
            }
         ?>
      </div> <!-- /container -->

      <div class = "container">

         <form class = "form-signin" role = "form"
            action = "<?php echo htmlspecialchars($_SERVER['PHP_SELF']);
            ?>" method = "post">
            <h4 class = "form-signin-heading"><?php echo $msg; ?></h4>
            <input type = "text" class = "form-control"
               name = "username" placeholder = "kullanici adi"
               required autofocus></br>
            <input type = "password" class = "form-control"
               name = "password" placeholder = "parola" required>
            <button class = "btn btn-lg btn-primary btn-block" type = "submit"
               name = "giris yap">Giris </button>
         </form>



      </div>

   </body>
</html>
```

#### Dosya İzinleri ve Sahiplikler

	$ls -la s36ozgur.com/public_html/wordpress/
	drwxr-xr-x.  5 apache apache  4096 Aug 26 16:32 .
	drwxr-xr-x.  3 apache apache    39 Aug 26 16:15 ..
	-rw-r--r--.  1 apache apache   543 Aug 26 16:32 .htaccess
	-rw-r--r--.  1 apache apache   405 Aug 26 15:25 index.php
	-rw-r--r--.  1 apache apache 19915 Aug 26 15:25 license.txt
	-rw-r--r--.  1 apache apache  7346 Aug 26 15:25 readme.html
	-rw-r--r--.  1 apache apache  7165 Aug 26 15:25 wp-activate.php
	drwxr-xr-x.  9 apache apache  4096 Aug 26 15:25 wp-admin
	-rw-r--r--.  1 apache apache   351 Aug 26 15:25 wp-blog-header.php
	-rw-r--r--.  1 apache apache  2328 Aug 26 15:25 wp-comments-post.php
	-rw-r--r--.  1 apache apache  2989 Aug 26 15:40 wp-config.php
	drwxr-xr-x.  5 apache apache    67 Aug 26 17:26 wp-content
	-rw-r--r--.  1 apache apache  3939 Aug 26 15:25 wp-cron.php
	drwxr-xr-x. 25 apache apache  8192 Aug 26 15:25 wp-includes
	-rw-r--r--.  1 apache apache  2496 Aug 26 15:25 wp-links-opml.php
	-rw-r--r--.  1 apache apache  3900 Aug 26 15:25 wp-load.php
	-rw-r--r--.  1 apache apache 45463 Aug 26 15:25 wp-login.php
	-rw-r--r--.  1 apache apache  8509 Aug 26 15:25 wp-mail.php
	-rw-r--r--.  1 apache apache 22297 Aug 26 15:25 wp-settings.php
	-rw-r--r--.  1 apache apache 31693 Aug 26 15:25 wp-signup.php
	-rw-r--r--.  1 apache apache  4747 Aug 26 15:25 wp-trackback.php
	-rw-r--r--.  1 apache apache  3236 Aug 26 15:25 xmlrpc.php

	ls -la zubizu.com/public_html/
	total 4
	drwxr-xr-x. 3 apache apache  38 Aug 26 18:28 .
	drwxr-xr-x. 3 apache apache  25 Aug 26 12:44 ..
	-rw-r--r--. 1 apache apache 262 Aug 27 10:02 index.php
	drwxr-xr-x. 2 apache apache  23 Aug 26 18:28 yonetim

