S36 Teknik Sorun Çözme Mülakatı
===========================

#### Makinenin Çalıştırılması

İndirdiğim .ova dosyasını Oracle VirtualBox uygulamasına import ettim. Başlattığımda network ayarlarından dolayı çalışmadı. Sanal makinenin oluşturulduğu makinedeki interface ismi ile benim host makinemdeki interface ismi eşleşmediğinden ilerleyemedim. Host makinemdeki virtual interface ismini .ova sanal makinesindeki ile değiştirdim.

#### Root Parolasının Sıfırlanması

Makineyi çalıştırmamın ardından giriş bilgilerine sahip olmadığımı anladım. Basit kullanıcı adı parolalar denesem de başarılı olamadım. Dolayısıyla root parolasını grub üzerinde sıfırlamak için işlemlere başladım.

Makineyi yeniden başlatıp grub menüsüne girmek için e (edit) tuşuna bastım.

Ardından kernel satırında ro (read only olan dosya sistemini) rw(read write) olarak değiştirdim(Rw şekilde mount etmemizin sebebi parolayı değiştirip dosya sistemine yazabilmek.). Sonrasında tüm işletim sistemini başlatacak ve tüm processlerin atası olan init komutuna system'i başlatmak yerine /sysroot/bin/sh'ı çalıştırması gerektiğini belirttim. 

Ctrl +x ile çalıştırıp emergency shell'in açılmasını bekledim. Emergency shell bu tür bakım ve kurtarma işlemlerinin yapılabilmesi için basit komutlar çalıştırabilen bir ortam. Bu ortam gerçek filesystem'da olmadığı ve sistemin ayağa kalkması sırasında çalışan bir shell olduğundan root dizinini sistemin root dizini olarak değiştirdim. 
	
	chroot /sysroot

Buradan sonrasında root yetkilisi olarak dosya sisteminde bash komutları çalıştırabilecek hale geldim. Sonrasında
	
	passwd

komutuyla root parolasını sıfırlayıp yenisini oluşturdum. Centos Selinux ile çalıştığından relabel yapabilmesi için 

	touch /.autorelabel

komutunu çalıştırdım. Bu da Selinux tüm dosya sistemini tekrardan etiketlemesini sağlıyor. Böylece tüm dosyaların izinleri tekrardan oluşturulur.

	reboot 

komutunu çalıştırarak işletim sisteminin açılmasını bekledim.
Yeni oluşturduğum root parolasıyla giriş yaptım.

#### Apache'nin Ayağa Kaldırılması
Kök dizindeki readme dosyasını okuduktan sonra Apache'nin çalıştırılması gerektiğini anladım. Apache'nin durumunu sorgulamak için
	
	$systemctl status httpd

komutunu çalıştırdım. Apache'nin inaktif olduğunu gördüm. 
Çalıştırmak için

	$systemctl start httpd

komutunu çalıştırdım. Hata verdi ve servis çalışmadı. Configde hata olabildiğine inandığımdan

	$apachectl configtest

komutunu çalıştırdım. Conf dosyasında hata olduğunu gördüm. /etc/httpd/logs dizini yoktu veya erişilemiyordu. Dosya dizininde /etc/httpd/logs dizinine gidince başka bir dizine bağlı olduğunu gördüm. Bağlı dizin oluşmadığı için /etc/httpd/logs dizinin ucu bir yere çıkmıyordu. Ben de o dizini oluşturdum.

	$mkdir /var/log/httpd

Sonrasında tekrardan apache configtest yaptım ve Syntax OK geri dönüşünü aldım. Apache servisini başlattım. Apache çalıştı. Host makinemden web sayfasını kontrol ettim. Host makinemden web sayfasına ulaşamayınca sanal makinenin firewall ayarlarını inceledim. http servisinin dışarıya açık olmadığını gördüm ve izin verdim.

	$firewall-cmd --zone=public --permanent --add-service http
	$firewall-cmd --reload

Host makinemden tekrar ipye istek atınca bu sefer bağlanabildiğimi lakin içeriği görmek için doğru izinlere sahip olmadığımı gördüm.

index.html dosyasının kullanıcı ve grup sahipliğini apache yaptım.

	$chown apache:apache index.html

Tekrar test ettiğimde sayfayı görüntüleyebildim. Apache'yi güncellemem gerektiğine dair bir not gördüm. 

Apache'yi güncellemeden önce conf dosyalarını inceledim. Özel bir şey göremedim lakin yine de yedeklerini almaya karar verdim. 

#### Cpu, Ram ve Disk İyileştirmeleri

Yedek almaya çalışırken dosya sisteminde yer kalmadığı uyarısıyla karşılaştım. Sonrasında apache backup ve update işlemini erteleyip sistemin ana bileşenlerini kontrol etmeye başladım(cpu,ram,disk). Bazı komutların yavaş çalıştığından da belli olduğu gibi arkada ram ve cpu kullanan bir işlem olduğunu anladım. 

	$top

komutunu çalıştırıp ram'e göre sıraladıktan sonra sunucunun stress testi altında olduğunu gördüm. Stres işlemi belli aralıklar ile test ortamında veyahut canlı ortamda da yapılabilir. Lakin çalışmamı engelleyecek seviyede olduğundan stress işlemini kapadım.

	$kill 655

Ram ve cpu üzerindeki yük kalktı ve sunucu daha hızlı hale geldi.

Ardından dosya sisteminde yer kalmadı hatası ile ilgilenmeye başladım. 

	$df -h 

komutunu çalıştırdığımda diskte yer olmadığını farkettim. Dosya sisteminde neresinin dolu olduğunu aramaya başladım. 

README dosyasında en az yarım saat çalışır halde bırakın yazınızı düşündüm. Zamana bağlı bir şeyler koymuş olmalıydınız ki yarım saatin sonunda bu görev, bir veya birden fazla kere çalışacak ve bir şekilde sunucunun sağlığını bozacaktı. Eğer bir script zamana bağlı çalıştırılmak istenirse cronjob yazılması gerektiğini biliyordum. Bu yüzden 

	$cronjob -l

komutunu çalıştırarak zamanlanmış görevi buldum. 

	#Ansible: script2
	* * * * * /opt/.script > /var/www/html/index.html

Burada "Apache sürümünü güncelleyin" metnini, bir if koşuluna bağlı olarak (Apache'nin en güncel verisyonuna eşit olması) index.html'in içine yazdıran bir script çalışıyordu.

Bu script2 ise script1 de olmalıydı ama crontab - l denildiğinde bulunamıyordu. Script2'nin bulunduğu dizine gittim ve script1'i buldum. Dosya sisteminde kopyalama yapan bir script idi.
	
	dd if=/dev/zero of=/var/tmp'date +%s' .bin bs=1M count=1063

Kopyalama yaptığı dizine gittim.

	/var/tmp/

Bu dizinde 5 dakikada bir oluşturulmuş yaklaşık 1gblık dosyalar buldum. Zamanlanmış bir görev olduğundan böylece emin oldum. script1'in arkada çalışan bir servisin içine gömülebileceğini düşündüm.

	$systemctl -a

Lakin ilgi çekici ya da şüpheli bir şey bulamadım. Sonrasında aklıma crontab dosyasının en sonuna da cronjob yazılabileceği geldi. Crontab dosyasını açınca içinde diski dolduran scriptin 5 dakikada bir çalıştırılmak üzere ayarlandığını gördüm.

	$nano /etc/crontab

	#Ansible: script1
	*/5**** root /opt/.script1

Açıkçası bunun zararlı bir script mi yoksa yanlış konfigure edilmiş bir yedek/bakım script'i mi olduğunu çözemedim. Elimde yedek olduğuna dair bir bilgi olmadığından cronjob listesinden sildim.

	$rm -rf *.bin

komutunu çalıştırarak oluşturulmuş 1er gblık binary dosyaları da sildim.

Cpu,ram ve disk üzerindeki iyileştirmelerim bittiğinden Apache update işine geri döndüm.

#### Apache Update Yapılması

Diskte yer açtığımdan Apache conf dosyasının backupını aldım. Öncesinde backups dizini oluşturdum.

	$mkdir /var/backups

	$cp /etc/httpd/conf/httpd.conf /var/backups

Backup oluşturduktan sonra 

	$yum update

komutunu çalıştırdım. Lakin hiçbir mirror'a bağlanamadığını gördüm. Sonrasında internet bağlantısını kontrol etmek için

	$ping 8.8.8.8

komutunu çalıştırdım. Ip ye ping atabildiğime göre dns'de bir problem vardı. 

	$cat /etc/resolve.conf 

komutunu çalıştırıp dns'i kontrol ettim ve 8.8.8.7 olduğunu gördüm. 

	$nano /etc/resolve.conf 

komutunu çalıştırdım. Dosyanın root yazmasına korumalı olduğunu gördüm. 
	
	$chattr -i /etc/<resolve.conf

komutunu çalıştırıp dosyanın korumasını kaldırdım. Nameserver'ı 8.8.8.8 olarak güncelledim. Dosyayı tekrar korumalı hale getirdim.

	$chattr +i /etc/resolve.conf
	$yum clean all 
	

çalıştırarak yum cache'ini temizledim. Ardından
	
	$yum update 
	$yum install epel-release
komutlarını çalıştırdım.

Sonra Centos-Base.repo dosyasına gidip enabled=0 olan değeri 1'e çevirdim.

	 .
	 enabled=1
	 .

	$yum install httpd

diyip apacheyi güncelledim. Host makinesinden web sitesine bakınca apache güncel ve çalışıyor yazısını gördüm.

#### Son Test

Sunucu 1 saate yakın problemsiz çalıştı. Kapatıp açınca stress testinin tekrar başladığını farkettim, Service haline çevrilmiş bir bash scripti olduğunu saptadım ardından stress servisini disable edip kapadım.

	$systemctl disable stress
	$systemctl stop stress


	
#### /etc ve /root dizinlerinin kopyalanması

/etc ve root dizinlerini /var/backups dizinin altına kopyaladım.
	
	$cp -r /etc/ /var/backups
	$cp -r /root/ /var/backups
	$tar czvf etc_root.tar.gz etc/ root/

İki dosyayı zipledim ve host makinemden secure copy ile kopyaladım.

	$scp root@192.168.56.52:/var/backups/etc_root.tar.gz /home/kubison/


#### Sonuç ve Değerlendirme

Problem çözme mülakatında teknik kuruluma göre daha çok keyif aldığımı belirtmek isterim. Bir noktada gzip kullanıp /etc/ ve /root klasörlerini ziplemeye çalışırken /etc klasörüne zarar verdim. Hata yapma riskine karşın, ara adımlarımda aldığım sanal makine klonuna dönüp bir kaç adımı tekrarlayarak mülakatı tamamladım.
