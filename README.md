# Grid Up Datathon: Trafo Bazlı Elektrik Tüketimi Tahmini

Gdz ve Adm Elektrik Dağıtım A.Ş. tarafından düzenlenen Grid Up Datathon çözümü.

**Problem:** Trafo bazlı günlük elektrik tüketimini tahmin etmek.
**Metrik:** RMSLE (düşük skor daha iyi).
**Veri:** Ocak 2025 ile Mart 2026 arası eğitim (1.226.237 satır), Nisan ile Temmuz 2026 arası test (714.688 satır).

## Çözümün özeti

LightGBM ile direct forecasting. Hedef `log1p(tuketim)`. Modelin tahmini, trafonun kendi son 30 günlük ortalamasıyla log uzayında 70/30 harmanlanıyor.

Model 25 özellik kullanıyor: takvim bilgileri, lokasyon hiyerarşisi, kurulu güç, shrinkage uygulanmış grup istatistiği, trafo profili (ortalama, medyan, log istatistikleri, sıfır oranı, son 30 ve 90 günün özetleri), hafta günü profili ve 7 günlük ortalama sıcaklık.

Eğitim altı origin üzerinden yapılıyor. Üçü yaz hedefli (Nisan, Mayıs, Haziran 2025), üçü kış hedefli (Aralık 2025, Ocak ve Şubat 2026).

## Klasör yapısı

```
data/raw/          Kaggle'dan indirilen ham veri
data/external/     Open Meteo sıcaklık verisi ve iklim normalleri
notebooks/         Keşif, deney ve model notebookları
submissions/       Kaggle'a gönderilen tahmin dosyaları
```

Notebooklar:


| Dosya                           | İçerik                                         |
| ------------------------------- | ---------------------------------------------- |
| `01_EDA.ipynb`                  | Keşifsel veri analizi, veri kalitesi bulguları |
| `02_model.ipynb`                | İlk direct forecasting pipeline'ı              |
| `03_deneyler.ipynb`             | Hava durumu API çekimi ve çeşitli denemeler    |
| `04_ikili_model.ipynb`          | Cold start için ayrı model denemesi            |
| `05_kurguA.ipynb`               | Origin seti varyasyonları                      |
| `06_arastirma.ipynb`            | Cold start araştırması                         |
| `07_mevsim.ipynb`               | Mevsimsellik denemeleri                        |
| `08_gecen_yil_ayni_donem.ipynb` | Geçen yıl aynı dönem özelliği (elendi)         |
| `09_hava_durumu.ipynb`          | Hava durumu özellik seçimi                     |
| `10_final_model.ipynb`          | Final model için doğrulama ve ayar seçimi      |
| `11_final_submission.ipynb`     | **Final model ve submission üretimi**          |




## Veri seti

Yarışma verisi Gdz ve Adm Elektrik Dağıtım A.Ş. bölgesindeki dağıtım trafolarının günlük tüketimini içeriyor.


| Dosya                   | Satır     | Sütunlar                                       |
| ----------------------- | --------- | ---------------------------------------------- |
| `train.csv`             | 1.226.237 | `tanim`, `guc`, `tarih`, `tuketim`, `lokasyon` |
| `test.csv`              | 714.688   | `id`, `tanim`, `guc`, `tarih`, `lokasyon`      |
| `sample_submission.csv` | 714.688   | `id`, `tuketim`                                |



| Sütun      | Anlamı                                                                                 |
| ---------- | -------------------------------------------------------------------------------------- |
| `tanim`    | Trafo kimliği (eğitimde 5.344, testte 7.036 farklı trafo)                              |
| `guc`      | Kurulu güç. Medyan 400, 99. yüzdelik 10.900, en büyük 35.900                           |
| `tarih`    | Gün. Eğitim 1 Ocak 2025 ile 31 Mart 2026, test 1 Nisan 2026 ile 31 Temmuz 2026         |
| `tuketim`  | Günlük tüketim (hedef). Medyan 1.075, 90. yüzdelik 5.064                               |
| `lokasyon` | `İL>BÖLGE>İLÇE` hiyerarşisi. 2 il (İzmir yüzde 73, Manisa yüzde 27), 20 bölge, 30 ilçe |


Test, eğitimden hemen sonra başlıyor ve 122 gün ileriye uzanıyor. Aylık dağılım Nisan 117.604, Mayıs 182.673, Haziran 203.279, Temmuz 211.132 satır. Trafo sayısı zamanla arttığı için test ayları ilerledikçe büyüyor. Hedefte ham değerler çok çarpık olduğu için model `log1p(tuketim)` üzerinde çalışıyor ve metrik RMSLE.

Eğitim verisinde en büyük `tuketim` değeri 50.403.051, medyan 1.075. Bu fark fiziksel olarak imkansız kayıtlardan geliyor, aşağıdaki veri kalitesi bölümünde detayı var.

Dış veri olarak Open Meteo'dan ilçe bazlı günlük sıcaklık (`hava_durumu.csv`), ek hava değişkenleri (`hava_ek.csv`) ve 2015 ile 2024 arası iklim normalleri (`iklim_normal_ham.csv`) çekildi. Bunlar da repoya alınmadı, `03_deneyler.ipynb` ve `09_hava_durumu.ipynb` ile yeniden üretiliyor. Final modelde yalnızca 7 günlük ortalama sıcaklık kullanıldı.

## Skor seyri

Kaggle skoru ilk gönderimde 1.13709, ardından 1.08157, 1.06168, 1.05487, 1.05382 ve final modelde 1.05008 oldu. Her adım tek değişkenli bir değişiklikti. Neden ve nasıl sonuç verdiğini aşağıdaki yöntemsel bulgular anlatıyor.

## Veri kalitesi bulguları

**Fiziksel olarak imkansız değerler.** 444 satır, kurulu gücün 24 saatlik teorik tavanını 2 ile 58 kat aşıyor. Bunların yüzde 78'i sadece 4 trafoda toplanmış. Genel bir istatistiksel aykırı değer temizliği (IQR, z skoru) bunları bulamaz veya gerçek büyük trafoları da siler. `guc * 24` sınırıyla tespit edip trafo bazlı medyanla düzelttik.

**Cold start.** Test setindeki 7036 trafonun 2024'ü eğitim verisinde hiç yok. Satır bazında test'in yüzde 22,2'si. Bu trafolar için elimizde sadece kurulu güç ve lokasyon var.

**Kademeli veri girişi.** Trafoların yalnızca yüzde 23'ü eğitim döneminin tamamında (455 gün) veri içeriyor, medyan 170 gün. Yüzde 77'sinde aralık içi boşluk yok, yani geç başlamış veya erken bitmişler. Muhtemelen kademeli sayaç kurulumu.

**Sıfır tüketim.** 298 trafo eğitim boyunca istisnasız sıfır. Bunların 234'ü test setinde de var. Sıfır günleri toplam kare hatanın yaklaşık yarısını üretiyor.

**Eksik değer yok.** Hiçbir sütunda eksik kayıt bulunmuyor.

**Mevsimsellik.** Temmuz medyanı 1683 kWh, Mayıs medyanı 822 kWh. Yaklaşık iki kat fark.

## Yöntemsel bulgular



### Rekürsif tahmin tuzağı

Test ufku 122 gün. Rekürsif lag kullanımında (dünkü değeri özellik olarak vermek) test'in ikinci gününden itibaren "dünkü değer" artık modelin kendi tahmini oluyor ve hata birikiyor. Bunu ölçtük:


| Ufuk           | Direct | Rekürsif |
| -------------- | ------ | -------- |
| 1 ile 30 gün   | 0.831  | 0.843    |
| 31 ile 60 gün  | 1.008  | 1.070    |
| 61 ile 90 gün  | 1.093  | 1.194    |
| 91 ile 121 gün | 1.150  | 1.247    |


Kısa ufukta rekürsif önde, ufuk uzadıkça direct kazanıyor. 122 günlük gerçek test ufkunda direct her kovada önde. Bu yüzden rekürsif yaklaşımı bıraktık.

Bu hata sessizce gerçekleşiyor, model hata vermiyor, sadece tahminler zayıflıyor. Fark etmenin yolu test setinde lag özelliklerinin NaN oranına bakmaktı: yüzde 99,2 çıkmıştı, ve 121/122 = 0,992 hesabı mekanizmayı doğruladı.

### Yerel doğrulama Kaggle skorunu öngörmüyor

Bu projede en çok vakit kaybettiren şey buydu. Aynı değişikliğin hem yerel hem Kaggle sonucunu bildiğimiz dört karşılaştırma:


| Değişiklik                            | Yerel tahmin      | Kaggle gerçek |
| ------------------------------------- | ----------------- | ------------- |
| sub_02 tarifi ile sub_10 tarifi farkı | +0.117 ile +0.340 | +0.0267       |
| Hava artı hiperparametre paketi       | −0.033 ile −0.186 | +0.0009       |
| Yaprak 31'den 15'e                    | −0.0139           | −0.00105      |
| `sic_7gun` eklemek                    | −0.0244           | −0.00374      |


Yön dörtte üç doğru, büyüklük istisnasız 6 ile 13 kat abartılı. Ters çıkan tek vaka birden fazla değişikliği paketleyen submission'dı.

Sebebi yapısal. Kurulabilen bütün doğrulamalarda profiller 2 ile 12 aylık, gerçek testte 15 aylık. Zayıf profil rejiminde model başka sinyallere muhtaç olduğu için her ekleme büyük görünüyor, gerçek testte trafonun kendi yakın geçmişi baskın olduğu için geri kalan marjinal kalıyor.



### Basitleştirme sürekli kazandırdı

Yaprak sayısını düşürmek üç kez üst üste skoru iyileştirdi (63, 31, 15). 122 günlük ufukta basit ağaçlar trafoya özel örüntüleri ezberleyemedikleri için daha iyi genelliyor. Doğrulamada eğri 7 ile 10 bandında dipleniyor, yani bu yön yaklaşık tükenmiş.

### Mevsimsel kapsama

`yilin_gunu` özelliğinin eğitim değerleri ile test değerleri arasında hiç örtüşme yoktu (eğitim 335 ile 365 ve 1 ile 90, test 91 ile 212). Ağaçlar dış değer tahmini yapamadığı için bu özellik test'te işlevsizdi. Çıkarınca skor iyileşti.

Sıcaklıkta böyle bir sorun yok, test aralığı (5,7 ile 31,3 derece) eğitim aralığının (−2,9 ile 34,7) içinde kalıyor.

### 2026 yazı 2025'ten serin geçti


|                                      | Temmuz 2025 | Temmuz 2026 |
| ------------------------------------ | ----------- | ----------- |
| Ortalama sıcaklık                    | 30,0        | 27,5        |
| Maksimum sıcaklık                    | 37,2        | 33,8        |
| Soğutma derece gün (22 derece eşiği) | 8,00        | 5,53        |


Sadece takvime bakan bir model "Temmuz Temmuz'dur" varsayıp yüksek tahmin ediyor. Sıcaklık özelliği eklenince model Temmuz tahminini aşağı çekti ve skor iyileşti.

### Cold start bilgi tavanında

Trafonun tüketim seviyesini kurulu güç ve lokasyondan tahmin etmeyi denedik, 5 katlı çapraz doğrulamada R² sadece 0.195. Mevcut modelimizin bu segmentteki RMSLE'si 1,86, seviye tahmininin RMSE'si ise 1,94. Yani model bu bilgilerden çıkarılabilecek her şeyi zaten çıkarıyor.

Denenen ve başarısız olan alternatifler bu bölümün altında listeli.

## Denenip elenen yaklaşımlar

Her biri ölçüldü ve katkı sağlamadığı için bırakıldı.

**Geçen yılın aynı dönemi.** Test dönemi Nisan ile Temmuz 2026, elimizde Nisan ile Temmuz 2025 var. Farklı pencereler denendi (tam aynı gün, ±3, ±7, ±15 gün). Hedefle korelasyon 0.82 gibi yüksek görünüyor ama trafonun genel seviyesi ve son 30 günü hesaba katıldıktan sonra kısmi korelasyon 0.04'e düşüyor. Taşıdığı bilgi zaten elimizde. Detay: `08_gecen_yil_ayni_donem.ipynb`

**Cold start için kümeleme.** Trafolar davranış şekline göre (haftalık ve aylık sapma profili) altı kümeye ayrıldı, sonra kurulu güç ve lokasyondan küme tahmini denendi. Doğruluk 0.791, hep en büyük kümeyi söylemek 0.799. Rastgeleden kötü.

**Trafo ID önekleri.** ID'lerin fider veya dağıtım merkezi kodu taşıyabileceği düşünüldü. Ham ilişki umut verici (önek5 için eta kare 0.175, ilçenin 0.090'ından iyi) ama çapraz doğrulamada zarar verdi (R² 0.192'den 0.172'ye). 219 kategori, her birinde ortalama 21 trafo, model ezberliyor.

**ID sayısal komşuluğu.** Sayısal olarak en yakın trafoların ortalama seviyesi. Tek başına R² 0.065 ile 0.094, kurulu güç ve lokasyona eklendiğinde R²'yi 0.201'den 0.181'e düşürüyor.

**Hep sıfır trafolar için tahmin sıfırlama.** Eğitim boyunca istisnasız sıfır olan 241 trafonun test tahminleri sıfıra ezildi. Aritmetik hesap kazanç gösteriyordu ama Kaggle skoru 1.05008'den 1.18446'ya çıktı. Geriye dönük analiz, bu trafoların Nisan ile Temmuz 2026 döneminde büyük ölçüde enerjilendiğini ve modelin yüksek tahminlerinin doğru olduğunu gösteriyor. Eğitim verisinden ölçtüğümüz yeniden enerjilenme oranı (yüzde 3 ile 7) Ocak ile Mart penceresinden geliyordu, ilkbahar ve yaz farklı davranıyor.

**Genişletilmiş hava durumu değişkenleri.** Hissedilen sıcaklık, güneş radyasyonu, güneşlenme süresi, referans evapotranspirasyon (tarımsal sulama talebi göstergesi), yağış, rüzgâr. Hiçbiri katkı sağlamadı, hepsi tek sıcaklık sütunundan biraz daha kötü. Detay: `09_hava_durumu.ipynb`

**Sıcaklık anomalisi.** 2015 ile 2024 arası on yıllık iklim normalleri çekildi, anomali (gerçekleşen eksi normal) hesaplandı. Ham sıcaklığın yerine konduğunda sinyalin neredeyse tamamı kayboluyor (yaz doğrulamasında 1.0313'ten 1.1727'ye, yani hava durumu hiç olmayan seviyeye). Yanına eklendiğinde model hiç kullanmıyor. Sebebi fiziksel: klima yükünü mutlak sıcaklık belirliyor, normalden sapma değil.

**Trafo bazlı sıcaklık hassasiyeti.** Her trafonun soğutma derece güne tepki eğimi. Eğitim setinde ancak yüzde 7 doldurulabildi çünkü erken origin'lerin geçmişinde yaz yok, eğim hesaplanamıyor. Zarar verdi.

**Hedefi kullanım oranına çevirmek.** `log(tuketim)` yerine `log(tuketim/guc)`. Kurulu güç tüketim seviyesinin ancak yüzde 20'sini açıkladığı için offset olarak zayıf kaldı, doğrulamada 0.7844'ten 0.8333'e kötüleşti.

**Mevsimsel arındırma.** Kısa geçmişli trafoların ortalamaları mevsimsel olarak yanlı. Yanlılığı grup mevsim endeksiyle düzeltmeyi denedik. Mekanizma doğru ama büyüklüğü yetersiz: yanlılığın trafolar arası standart sapması sadece 0.073 log birimi, trafolar arası gerçek seviye farkları ise birkaç log birimi mertebesinde.

**Bayram rampası ve tatil listesi düzeltmesi.** Dini bayramlar her yıl yaklaşık 11 gün kayıyor. Bayrama uzaklık özelliği eklendi ve 2026 Kurban Bayramı'nın idari izin günleri (25 Mayıs dahil) listeye işlendi. Skor değişmedi. Sebebi feature importance tablosunda görünüyor: `tatil` özelliğinin bölünme sayısı sıfır, model bu sütuna hiç bakmıyor. Tatil boyutunun matematiksel tavanı yaklaşık 0.0003 RMSLE, çünkü tatil günleri test'in yüzde 6,75'i ve etki yaklaşık yüzde 10.

**Ölü özellikleri çıkarmak.** `il`, `tatil`, `haftanin_gunu`, `gun`, `hafta_sonu` özelliklerinin bölünme sayısı tam sıfır. Çıkarınca tahminler bit düzeyinde aynı kalıyor. LightGBM'de özellik bütçesi yok, `colsample_bytree` ayarlı olmadığı için her bölünmede tüm özellikler değerlendiriliyor.

**Grup bazlı mevsim endeksi özelliği.** Bölge ve güç kovası bazında aylık log sapma, shrinkage uygulanmış hali dahil. Model hiç kullanmadı, skorlar dört haneye kadar birebir aynı kaldı. Eğitim verisinde `bolge`, `guc_bucket` ve `ay` bölünmeleriyle aynı bilgi zaten kurulabildiği için gereksiz kalıyor.

**Artık modelleme.** Hedefi `log1p(tuketim)` yerine baz çizgisinden sapma olarak tanımlamak. Kış doğrulamalarında kazandırdı ama yaz geçişi doğrulamasında zarar verdi (1.0313'ten 1.1482'ye), testimiz o yapıya benzediği için bırakıldı. Aynı fikrin özellik yerine harman olarak uygulanması final modelde kullanıldı.

## Özellik önemi

Kazanç bazlı, final modelden:


| Özellik          | Kazanç payı |
| ---------------- | ----------- |
| `son30_log_ort`  | yüzde 78,9  |
| `son90_log_ort`  | yüzde 5,5   |
| `trafo_medyan`   | yüzde 3,3   |
| `guc`            | yüzde 2,1   |
| `trafo_ortalama` | yüzde 1,8   |
| `sic_7gun`       | yüzde 1,7   |
| `ilce`           | yüzde 1,3   |
| Kalan 18 özellik | yüzde 5,4   |


Permütasyon testinde `son30_log_ort` karıştırıldığında skor 1.74 bozuluyor, ikinci sıradaki özellik 0.06'da kalıyor. Model esasen trafonun son 30 gün ortalamasını alıp üzerine düzeltme yapan bir yapı. Final modeldeki baz çizgisi harmanı bu gözlemden çıktı.

`sic_7gun` için bir not: yukarıdaki permütasyon testi kış dönemi doğrulamasında yapıldı ve orada sıcaklık dar bir aralıkta oynadığı için önemi düşük görünüyor. Bölünme sayısında üçüncü sırada ve Kaggle'da eklenmesi 0.0037 kazandırdı.

## Kalan sınırlar

Test satırlarının yüzde 22'si eğitim verisinde hiç görünmeyen trafolara ait. O segmentte elimizde sadece kurulu güç ve lokasyon var, bu ikisi tüketim seviyesinin yüzde 20'sini açıklıyor, ve segment RMSLE'si 1,86 civarında. Toplam skorun büyük kısmını bu belirliyor ve veride bulunmayan bilgiye bağlı.

Sıfır tüketim günleri satırların yüzde 4,7'si ama toplam kare hatanın yaklaşık yarısını üretiyor. Bunların çoğu sporadik arıza günleri ve statik özelliklerle öngörülemiyor.

Eğitim verisinde Nisan ayı sadece 64 bin satır, Temmuz 227 bin. Test'in dörtte biri Nisan ve model o ayı en az görüyor. Veride tek bir Nisan var (2025) ve o dönemde henüz az trafo raporluyordu.

## Kurulum

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

