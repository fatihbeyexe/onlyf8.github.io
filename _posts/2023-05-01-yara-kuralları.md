---
title: YARA Kuralları
language: TR
published: true
category: malware
---

<h1 style="text-align:center">İçerik </h1>

+ YARA kuralı nedir?
+ Meta
+ Stringler
+ Condition'lar
+ Modüller

---

<h1 style="text-align:center"> YARA Kuralı Nedir? </h1>

YARA, belirli bir syntax'a sahip olan, zararlı yazılımları sınıflandırma/tanımlama aşamasında analistlerin kullandığı bir araçtır. Bu yazılım için yazılmış olan kurallara da YARA Kuralı denir. Bu blog yazısı genel konsept olarak YARA kurallarının nasıl yazıldığı/okunduğu hakkında bilgiler içermektedir.

C diline benzer bir yazım şekli vardır. Belirli anahtar kelimeleri bilmek YARA kuralı okuma esnasında bize yeterli olacaktır. Tabii ki bir kural yazmak için bilgi ve tecrübe gereklidir. Yani zamanla gelişecek bir kavramdır. Peki bu YARA kurallarının kullanım amaçları nedir? 

1. Bir zararlı yazılımın hangi zararlı yazılım ailesine ait olduğunu tespit etmek
   > İncelenen sample hakkında yazılmış farklı raporlar okunarak daha hızlı analiz gerçekleştirilebilir
2. Olay müdahalesi esnasında çeşitli IOC tarama araçları (THOR, Loki vs.) yardımıyla YARA Kural setleri kullanarak sistem üzerindeki zararlı yazılımların varlıklarını tespit etmek
3. Belirli bir zararlı yazılım ailesine ait yeni çeşit bir sample tespit etmek
4. Proaktif güvenlik için kullanmak, 

Genel olarak yukarıdaki 4 amaç söylenebilir. Spesifik kullanımlarda farklı amaçlara hizmet edebilir.

Peki bu kural dosyası neleri içerebilir/içermelidir? Buradaki önemli nokta şudur; yazacağımız kural'ın false positive üretme ihtimalini en düşük seviyede tutmamız gerekmektedir. False positive, güvenlik ürünü açısından bakıldığında; bir zararlı yazılım tespit alarmı üretildi fakat alarma sebep olan yazılım legal. Bu duruma "false positive" yani yanlış pozitif denmektedir. Eğer bir kural fazlaca false positive üretiyorsa, analistlerin işini oldukça zorlaştırmaktadır bu yüzden yazılan kurallarda ana odak noktamız minimum false positive olmalıdır. Bunun için ne yapmalıyız? Bir yazılımı eşsiz kılan özellikler üzerinden kural yazmamız gerekmektedir. Örneğin kuralı "GetProcAddress, socket ve recv" stringleri geçen yazılımlara alarm ver şeklinde oluşturursak, legal olan yazılımlara da alarm üretilme ihtimali vardır, aynı zamanda bu 3 değer ile herhangi bir zararlı yazılım ailesine ait olup olmadığını tespit edemeyiz. O yüzden, hem o zararlı yazılım için spesifik olan kurallar hem de -eğer sınıflandırma için kullanılacaksa- o zararlı yazılım ailesi için spesifik olan kurallar yazılmalıdır. 

Kural dosyaları string veya binary patternlerden oluşmaktadır. Örneğin hard-coded şekilde bulunan bir C2 IP adresi kural içerisine yazılabilir. Bu değer dosya içerisinde varsa alarm üret şeklinde bir kural oluşturulabilir. 

---

<h1 style="text-align:center"> Meta </h1>

Yazılan kural içerisinde bulunan kısımlardan birisi **Meta** kısmıdır. Kuralın işlevselliği üzerinde etkisi yoktur. Genel olarak şunları içermektedir :

1. **Yazar**: Kuralın yazan kişinin ismi veya nickname'i
2. **Tarih**: Kuralın yazılma tarihi
3. **Referans**: Kuralın yazılma sebebi olan zararlı yazılımı veya tekniği tanımlayan bir blog yazısı vb. referanslar verilmektedir. 
4. **Açıklama**: Bu kuralın **neden** yazıldığı bilgisi bulunur. Örneğin **"detects mimikatz"**.
5. **Hash**: Kural yazılırken kullanılan sample'ların hash bilgileri bulunur.

Toplu şekilde IOC taramalarında meta kısmı oldukça önemlidir. Bir eşleşme olması durumunda analistin, bu eşleşmenin neden olduğuna dair bilgi edinmesi için gereklidir. Örneğin **"test.abc"** isimli bir dosya için **"detect mimikatz"** isimli bir kural tarafından alarm üretildiğinde bu alarmın neden üretildiği ve kimin yazdığı hakkında bilgileri analiste sunar.

```
rule test_with_yara{ // her bir kuralın başlangıcında "rule" anahtar kelimesi bulunur, ardından bu kuralın ismi ve süslü parantezler içerisine kural yazılır.

    meta:
        Author="OnlyF8"
        Date="01-05-2023"
        Description="Testing with YARA"
}
```

Şeklinde bir meta kısmı tanımlanabilir. Burada kullanılan "Author, Date" vb anahtar kelimeler ön tanımlı değildir. Yani istediğiniz herhangi bir isim verebilirsiniz. 

---

<h1 style="text-align:center"> String'ler </h1>

String kısmında, o zararlı yazılımı veya zararlı yazılım ailesini spesifik olarak tanımlayan IOC'lerin yazıldığı kısımdır. Buradaki değişkenler; hexadecimal, düz string veya regex olabilir. 

# Hexadecimal String'ler

Yazılımın ham halinde bulunan ve okunabilir olmayan dizeler için kullanılmaktadır. Örneğin XOR anahtarı veya UTF-8 olmayan dizeler için kullanılabilir. Burada 3 tür kullanım mevcuttur. Bunlar; wild-cards, jumps ve alternatiflerdir.

**Wildcard**'lar bir regex türü olarak düşünülebilir. Örnek olarak şu kurala bakalım: 

```
rule TestforWildcard{
    strings:
        $hex= { A1 29 ?? BA }
    condition:
        $hex
}

```
Örnekteki kuralda yazılan **$hex** dizesinde belirli bir kısım bilinmiyorsa veya değişkenlik gösteriyorsa bu şekilde tanımlanabilir. Burada soru işareti koyulan byte'lara bir nevi brute-force atılarak tarama yapılmaktadır.

**Jumps** ise bir örüntü tespit edildiğinde arasında belirli uzunluktaki byte değişikliği değil de belirgin olmayan bir değişiklik oluyorsa kullanılmaktadır. Örneğin bir zararlı yazılımda **"C:\ProgramData\Hello\abcd.exe"** dizesi geçiyor. Burada bulunan **"Hello"** dizini ve **"abcd.exe"** kısımları değişkenlik gösterebilir fakat uzunluğu sabit olmayabilir. Bundan dolayı jumps kullanılır. 

```
rule TestforJumps{
    strings:
        $jumps= { A1 29 [2-4] BA }
    condition:
        $jumps
}
```

Şeklinde yazılan bir kuralda ise belirtilen boyut aralığında brute-force atılmaktadır. Örneğin;

```
A1 29 00 00 BA
A1 29 00 01 BA
.
.
.
A1 29 00 00 00 AB BA
.
.
.
```

**Alternatives** yapısında ise, birbirinin alternatifi olan byte'lar taranmaktadır. Örneğin;

```
rule TestforAlternatives{
    strings:
        $alt= { A1 20 ( DC | 59 ) BA }
    condition:
        $alt
}
```

Şeklinde tanımlanan bir kuralda, **"A1 20 DC BA"** veya **"A1 20 59 BA"** bulunması durumları için kural oluşturulmuştur. Birden fazla alternatif ve bu alternatiflerin içerisinde jumps veya wild-cards kullanılabilir.

# Düz Yazılar

YARA kurallarında varsayılan olarak büyük-küçük harfe duyarlıdır. Yani bir kuralda **"onlyf8"** yazıyor fakat dosyanın içerisinde **"onlyF8"** olarak bulunuyorsa eşleşme olmamaktadır. 

Büyük-küçük harfe duyarlı olmaması için tanımlanan dizenin yanına **"nocase"** anahtar kelimesi yazılmalıdır.

Dosya içerisinde bulunan dizeler her zaman yazıldığı gibi kodlanmamaktadır. Bazen her bir karakter için 2 byte yer ayrıldığı durumlar olabilir. Bu gibi durumlar için **"wide"** anahtar kelimesi kullanılır. Örneğin **"Only"** dizesi dosya içerisinde **"O\x00n\x00l\x00y\x00"** şeklinde bulunabilir. 

Tek byte ile XOR'lanarak saklanmış verilen için **"xor"** anahtar kelimesi kullanılmaktadır. Örneğin;

```
rule TestforXOR{
    strings:
        $testxor="Onlyf8" xor(0x01-0xab)
    condition:
        $testxor
}
```
Yukarıdaki kural **"Onlyf8"** dizesinin tek byte (00, 01, 02 vs.) ile XOR'lanmış bütün durumlarını taramaktadır. XOR anahtar kelimesinden sonra parantez içinde belirtilen aralıkta XOR brute-force yapmaktadır. Bu özellik YARA 3.11'den sonra kullanılabilir hale gelmiştir.



## Base64 Değerler

Yazılan YARA kurallarında, yazılım içerisinde tespit edilen **Base64** ile encode edilmiş gömülü stringlerin decode edilmiş halleri bulunabilir. Örneğin aşağıdaki kural "OnlyF8" stringinin Base64 ile encode edilmiş halini dosyanın içerisinde tespit ettiğinde alarm üretmektedir.

```
rule TestforBase64{
    strings:
        $textbase64="Onlyf8" base64
    condition:
        $textbase64
}

```
## Fullword Taraması

**Fullword** anahtar kelimesi, alfanumerik olmayan karakterle ayrılmış stringleri tespit etmeye yaramaktadır. Örneğin **"hello"** isminde domain IOC değeri elde ettiniz. Eğer "hello" değeri "fullword" olarak tanımlanırsa; "www.hellobois.com" ile eşleşmeyecektir fakat "www.hello-bois.com" veya "www.hello.com" domainleriyle eşleşme sağlanacaktır.

---

**Anahtar kelime tablosu:**

|Anahtar kelime| String Türü| Açıklama|
|-|-|-|
|nocase|Text, Regex| Büyük-küçük harfe duyarlı değil|
|wide|Text, Regex| Dizeyi UTF-16 olarak yorumlar ve 0x00 durumlarını değerlendirir|
|ascii|Text, Regex| ASCII karakterleri kullanıldığında kullanılır (İngilizce olmayan karakterler için)|
|xor| Text| Verilen dizeyi tek byte ile XORlayarak tarama yapar|
|base64|Text| 3 ayrı Base64 permütasyonu ile tarama yapar|
|base64wide| Text| 3 ayrı Base64 permütasyonunu 0x00 ile birleştirerek tarama yapar|
|fullworld|Text, Regex|Verilen dizenin öncesinde veya sonrasında alfa numerik karakterler bulunmadığı durumlarda eşleşme olmasını sağlar |
|private|Hex, Text, Regex|Eşleşme, çıktıda gösterilmez|

---

# Koşullar

Koşullar basitçe bir mantıksal işlem olarak düşünülebilir. **"XOR"**, **"AND"**, **"OR"** gibi mantıksal operatörler kullanılabilmektedir. Genel olarak yazılım dillerindeki "IF" yapısı gibi çalışmaktadır. Örneğin;

```
rule APT34_Malware_HTA {
   meta:
      description = "Detects APT 34 malware"
      license = "Detection Rule License 1.1 https://github.com/Neo23x0/signature-base/blob/master/LICENSE"
      author = "Florian Roth (Nextron Systems)"
      reference = "https://www.fireeye.com/blog/threat-research/2017/12/targeted-attack-in-middle-east-by-apt34.html"
      date = "2017-12-07"
      hash1 = "f6fa94cc8efea0dbd7d4d4ca4cf85ac6da97ee5cf0c59d16a6aafccd2b9d8b9a"
   strings:
      $x1 = "WshShell.run \"cmd.exe /C C:\\ProgramData\\" ascii
      $x2 = ".bat&ping 127.0.0.1 -n 6 > nul&wscript  /b" ascii
      $x3 = "cmd.exe /C certutil -f  -decode C:\\ProgramData\\" ascii
      $x4 = "a.WriteLine(\"set Shell0 = CreateObject(" ascii
      $x5 = "& vbCrLf & \"Shell0.run" ascii
      $s1 = "<title>Blog.tkacprow.pl: HTA Hello World!</title>" fullword ascii
      $s2 = "<body onload=\"test()\">" fullword ascii
   condition:
      filesize < 60KB and ( 1 of ($x*) or all of ($s*) )
}
```
Yukarıdaki YARA kuralı örneğinin **"condition"** kısmına bakıldığında; "$x" ile başlayanlardan 1 tanesi veya "$s" ile başlayanların hepsinin bulunduğu durumlarda alarm üretecektir. Bu da bir zararlı yazılım ailesi veya bir saldırı grubunun kullandığı teknikleri bir kuralda toplamak için kullanılan tekniklerdendir.

Yazılan IOC değerleri (string, hex vb.) bir zararlı yazılım ailesini veya bir saldırı grubunu tespit etmek için yazılabilmektedir. Bu gibi durumlarda belirli IOC'ler hepsinde bulunurken bazı durumlarda değişiklik gösterebilmektedir. Bu gibi durumlarda farklı koşullar oluşturulmaktadır. 

"Condition" kısmında farklı YARA kuralı kütüphaneleri kullanılarak, dosya imzası vb. hexadecimal değerler de kontrol edilebilmektedir.


---

Eleştiri/düzeltme/öneri için lütfen iletişim adreslerimden bana ulaşınız. Yorumlarınız benim için değerli :)

---

# Referans

[1] https://yara.readthedocs.io/en/stable/
