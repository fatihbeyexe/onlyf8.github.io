---
title: Zararlı Doküman Analizi Part-1 (Microsoft Word Dokümanları)
language: TR
published: true
category: malware
---

<h1 style="text-align:center"> Zararlı Dokümanlar Nelerdir? </h1>

Siber saldırılarda bir saldırının birden çok aşaması bulunmakta ve bu aşamalar belirli amaçlar içermektedir. Cyber Kill Chain kavramında saldırganların başarılı bir saldırı yapmak için farklı aşamalarda farklı araçlar/teknolojiler kullandığı bilinmektedir. Bu yazımızda genellikle saldırıların **Exploit** aşamalarının başlangıcında kullanılan zararlı dokümanların nasıl tespit ve analiz edildiğini ele alacağız. 

Odak noktasını ve amacımızı belirlemek için zararlı dokümanlarda kullanılan teknikleri ve dilleri bilmemiz gerekmektedir. **Microsoft Office** dokümanlarına bakıldığında içerisinde; shellcode, VBA makroları ve embedded file'lar bulunabilir. PDF dosyalarında ise JavaScript, FlashScript, ActionScript ve embedded file'lar bulunabilir. XPS dosya türünde ise genellikle URI yönlendirme ile zararlı bağlantılardan dosya indirilmesi sağlanmaktadır fakat XPS dosyalarının kullanımı oldukça azdır. Bu tekniklerin yanı sıra yeni yöntemler de gün geçtikçe ortaya çıkabilmektedir. Herhangi bir dosya türü üzerinde yapılan zararlı yazılım analizinde 0-Day durumu göz önünde bulundurulmalıdır. Örneğin Microsoft Office dokümanları içerisinde daha önce görmediğiniz şekilde bir zararlı kod bloğu bulunabilir. Yukarıda bahsedilen örnekler her zaman için kesinlik taşımamaktadır. Bkz. Follina Zero Day.

Doküman analizinin kabaca adımları şu şekildedir:

```
1. Dosyaya gömülü olan kodu belirle (shellcode, script, executable)
2. Şüpheli kodu dosyadan ayıkla
3. Deobfuscation/decode (gerekiyorsa) işlemi yaparak kodu yorumlanabilir hale getir
4. Açık kaynaklı bir dil değilse disassemble ve debug yaparak kodu analiz et
5. Saldırının bir sonraki aşamasını belirle
```

Yukarıda bahsedilen tekniklerin bilinmesi zararlı dokümanların tespitinde bize yol göstermektedir. Örneğin elimizde şüpheli bir dosya olduğu senaryoda bakacağımız noktalar bu tekniklerin dosya içerisinde olup olmamasıdır. Bu noktada çeşitli araçları yardımcı olarak kullanacağız. 

---

<h1 style="text-align:center"> Office Dokümanları </h1>

97-2003 arası Microsoft Office dokümanları OLE2 formatında saklanmaktadır. 2003 ve sonrası Microsoft Office versiyonlarında XML formatına geçilmiştir. Bu yapısal fark gözle görülür şekilde karşımıza çıkmaktadır. Analiz aşamaları çok farklı olmasa da kullanılan araçlar bazı noktalarda farklılık göstermektedir. 

<img title="Farklılık" src="difference.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

OLE2 formatında bulunan dokümanlarda makro dosyaları **Macros** dizini altında bulunur, XML formatında bulunan dokümanlarda ise **word** dizini altında bulunur. Fakat iki türün de makro dosyaları **.bin** uzantılı ve aynı yapıda bulunduğu için, direkt olarak doküman dosyasını **7zip** aracı ile ayıklayıp, makro dosyasını alıp **OfficeMalScanner** veya **oletools** araçlarıyla okunabilir hale getirilebilir. Analiz aşaması olarak bakıldığında sonraki aşamalar aynı seyretmektedir. 

1. Dosyaya gömülü olan kodu belirle (shellcode, script, executable)

    <img title="Doküman İçeriği" src="doc-01.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

Dokümana bakıldığında klasik olarak kullanılan; aslında gizli olmayan bir içeriği gizli gibi gösterip kullanıcıdan makroları aktifleştirmesini istemektedir. Buradan yola çıkarak dosya içerisinde makro olduğunu düşünebiliriz. 7-zip gibi yardımcı araçlarla dosyayı ayıklayıp içeriğine baktığımızda **vbaProject.bin**(genellikle) veya farklı isimde **.bin** uzantılı dosya içerisinde makro kodunu bulabiliriz. 

2. Şüpheli kodu dosyadan ayıkla

<img title="Ayıklama" src="doc-02.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

<img title="Dizin İçeriği" src="doc-03.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

3. Deobfuscation/decode (gerekiyorsa) işlemi yaparak kodu yorumlanabilir hale getir

<img title="vbaProject.bin" src="macro-00.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

**vbaProject.bin** dosyası okunabilir halde değildir. Bu dosyayı okunabilir hale getirmek için **olevba** veya **OfficeMalScanner** aracı kullanılabilir. Aracı kullandığımızda ise şu şekilde bir sonuç elde ederiz.

<img title="OfficeMalScanner" src="doc-04.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

<img title="Obfuscated" src="macro-01.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

Artık elimizde okunabilir bir VBA makro kodu var. Ağır bir şekilde obfuscate edilmiş olan makro kodunu adım adım analiz etmemiz gerekiyor.

4. "Açık kaynaklı bir dil değilse disassemble ve debug yaparak kodu analiz et" aşamasını geçiyoruz. Çünkü elimizde bir script var.

5. Saldırının bir sonraki aşamasını belirle

Öncelikle oluşturulan değişkenler ve fonksiyonların isimlerini okunabilir hale getiriyoruz. Random şekilde oluşturulmuş bu isimler okumayı oldukça zorlaştırmaktadır. Makro içerisinde tanımlanan fonksiyonlara göz gezdirildiğinde, **Document_Open** fonksiyonu doküman açıldığı anda çalışacağı için bu fonksiyon üzerinde adım adım statik analiz yaparak kodun ne yapmaya çalıştığını çözeceğiz. 

<img title="Obfuscated Macro" src="macro-02.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

Program, akış şeması takip edildiğinde bizi **"zwxcnxajcshyp"** isimli fonksiyona getirmektedir. Burada 5 adet ne olduğu anlaşılmayan hardcoded değer ve sonrasında yapılan **Create** işlemi dikkat çekmektedir. Fonksiyon içerisinde **function6**'nın çokca çağrıldığı ve hardcoded değerleri bu fonksiyona parametre olarak geçildiği görülmektedir. Bir **decode/decrypt** fonksiyonu olduğu şüphesi ile bu fonksiyonu analiz ediyoruz. 

<img title="Obfuscated Macro" src="macro-03.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

**function6** içerisinden çağrılan **asciiTable**(ismi bu şekilde değiştirilmiştir) fonksiyonuna bakıldığında, dictionary türünde bir liste oluşturduğu ve bu liste içerisinde ascii tablosu benzeri yeni bir tablo oluşturulduğu tespit ediliyor.

<img title="Oluşturulan ASCII Tablosu" src="ascii-table.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

Ascii tablosunun bu şekilde tekrar oluşturulmasının sebebi ise şudur; sandbox benzeri ürünler dosya içerisinde geçen hardcoded hex/decimal değerleri otomatik olarak farklı türlere çevirip içerisinde şüpheli stringler aramaktadır. Saldırgan burada default ascii tablosunu kullanması durumunda bazı sandboxlar bu dosya için şüpheli/zararlı alarmı üretebilir. Bu yüzden bu şekilde yeni bir ascii tablosu oluşturulmaktadır.  

Ardından bu yeni oluşturulan ascii tablosundaki sayısal değerler kullanılarak, hardcoded olan **variable1,variable2,variable3,variable4 ve variable5** değerleri 3er basamak halinde parçalanarak çözümlenip, string haline dönüştürülmektedir. Burada yardımcı bir python scripti yazarak bu verilerin clear-text halini elde edebiliriz. 

<img title="Python Scripti" src="dict_to_string.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

Çözümlendiğinde ise aşağıdaki sonuçlar elde edilmiştir.

<img title="Obfuscated Macro" src="decoded_strings.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

<img title="Obfuscated Macro" src="macro-05.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

**Function5**'e bakıldığında **param1** değişkeninde verilen dosya isim ve dizininde bir dosya oluşturduğu anlaşılmaktadır.

<img title="Obfuscated Macro" src="macro-06.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

**Function9**'a bakıldığında ise IOC değerleri kolayca çıkartılmaması için oluşturulacak olan DLL dosyasının ismini rastgele oluşturulması için rastgele bir dosya ismi oluşturulmaktadır. 

Decode edilmiş değerleri elde ettiğimiz görselde yeşil ile kutucuk içerisine alınan satıra bakıldığında **CustomXMLParts** metodu ile **customXML** dizininde bulunan **XML** dosyaları içerisinde ismi **http://localhost/** olan değişkenin değeri **Text** olarak alınmaktadır. Bu değere bakıldığında ise içerisinde; byte sıralaması değiştirilip, tersi alınmış bir executable dosya bulunduğu tespit edilmektedir.

<img title="XML İçeriği" src="macro-04.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

İlgili veriler **CyberChef** aracı yardımıyla dosya haline getirilebilmektedir. Kullanılan tarif ise şu şekildedir;

<img title="CyberChef" src="cyberChef.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

Ardından elde edilen dosya indirildiğinde ise 64-Bit yapısında bir DLL olduğu tespit edilmiştir. 

<img title="SecondStage" src="secondStage.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

VirusTotal üzerinden yapılan sorguda ise IcedID ailesine ait bir **Downloader** olduğu tespit edilmektedir.

<img title="VirusTotal" src="virusTotal.png" style="display:block; margin-right:auto; margin-left:auto; padding-bottom:20px;">

Bu blog yazısının hazırlanmasında emeği geçen Mehmet DEMİR’e teşekkürler.

---

Eleştiri/düzeltme/öneri ve sorularınız için lütfen iletişim adreslerimden bana ulaşınız. Yorumlarınız benim için değerli :)
