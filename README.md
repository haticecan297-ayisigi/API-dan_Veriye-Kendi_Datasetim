# TVMaze API: Bölüm Puanları Üzerine Veri Analizi

Bu proje, TVMaze API üzerinden toplanan TV dizi bölümü verilerini temizleyip keşifsel veri analizi ile incelemektedir. Veri; ham JSON biçiminde saklanmış, temizlenmiş CSV dosyasına dönüştürülmüş ve araştırma sorularına yönelik görseller oluşturulmuştur.

## Problem

TV dizilerinin sezonları ilerledikçe bölüm puanlarının değişimi, türlere göre puan dağılımlarındaki farklar ve bölüm süresi ile izleyici puanı arasındaki ilişki incelenmiştir.

### Araştırma soruları

1. Belirlenen dizilerde sezonlar ilerledikçe ortalama bölüm puanı düşüş eğiliminde mi?
2. Birincil türlere göre bölüm puanları, medyan değerleri ve varyansları bakımından nasıl farklılaşmaktadır?
3. Bölüm süresi ile bölüm puanı arasında anlamlı bir ilişki var mı?

## Veri

- **Kaynak:** [TVMaze API](https://api.tvmaze.com)
- **Veri çekme yöntemi:** Dizi arama, dizi bilgisi ve bölüm uç noktaları kullanılarak 12 dizi için bölüm verileri çekildi.
- **Ham veri:** `data/raw/tvmaze_raw.json`
- **Temiz veri:** `data/processed/tvmaze_episodes_clean.csv`
- **Ham kayıt sayısı:** 2572 bölüm
- **Temiz kayıt sayısı:** 2566 bölüm
- **Temiz veri sütun sayısı:** 19
- **Verinin incelenen ana alanları:** Dizi adı, dizi türü, türler, sezon, bölüm numarası, bölüm adı, bölüm süresi, bölüm puanı, yayın tarihi ve bölüm URL'si

Analizde kullanılan diziler:

- The Simpsons
- Doctor Who
- Friends
- Grey's Anatomy
- Supernatural
- The Office
- Brooklyn Nine-Nine
- Game of Thrones
- Breaking Bad
- Stranger Things
- Black Mirror
- Sherlock

Araştırma sorusu 1 için sezon trendlerinin karşılaştırılabilmesi amacıyla özellikle uzun soluklu beş dizi seçilmiştir: The Simpsons, Doctor Who, Friends, Grey's Anatomy ve Supernatural.

## Yöntem

### Veri toplama

Notebook, TVMaze API ile dizi araması yaparak tam isim eşleşmesi bulunan dizilerin:

- Dizi ayrıntılarını,
- Tüm bölüm kayıtlarını,
- Dizi türü, süre, puan ve sezon bilgilerini

çekmektedir. İstekler hata durumunda en fazla üç kez yeniden denenmektedir ve API'ye istekler arasında kısa bekleme süresi bırakılmaktadır.

### JSON'dan DataFrame'e dönüştürme

Ham JSON yanıtları Python sözlüklerinden oluşturulan kayıtlar aracılığıyla DataFrame'e dönüştürülmüş; tür, süre, puan, sezon ve bölüm kimlikleri uygun sayısal alanlara ayrılmıştır. Birincil tür olarak dizinin tür listesindeki ilk tür seçilmiştir.

### Veri temizleme

Temizlik aşamasında:

- `episode_id` alanına göre tekrar eden kayıtlar kontrol edildi ve temiz veri için tekrar eden kayıtlar kaldırıldı.
- Sayısal alanlar `pd.to_numeric` ile dönüştürüldü.
- Yayın tarihleri `pd.to_datetime` ile tarih biçimine dönüştürüldü.
- Puanı 0-10 aralığında olmayan geçersiz değerler eksik olarak işaretlendi.
- Süresi sıfır veya negatif olan geçersiz değerler eksik olarak işaretlendi.
- Dizi adı, bölüm kimliği, sezon ve bölüm puanı zorunlu alan olarak alınmış ve bu alanlardan biri eksik olan 6 kayıt temiz veri setinden çıkarılmıştır.
- Süresi eksik olan bölüm kayıtları genel temiz veri setinde tutulmuş; yalnızca bölüm süresi ve puan korelasyonunda süresi eksik kayıtlar pairwise olarak dışarıda bırakılmıştır.
- IQR yöntemiyle puan ve süre aykırı değerleri işaretlenmiş, analiz için silinmemiştir.

Eksik değerler, aykırı değerler ve temizleme adımları aşağıdaki rapor dosyalarında tutulmaktadır:

- `data/processed/cleaning_report.csv`
- `data/processed/missing_values_before_cleaning.csv`
- `data/processed/missing_values_after_cleaning.csv`
- `data/processed/outlier_report.csv`

## Bulgular

### Araştırma 1: Sezonlara göre puan eğilimi

Sezon ortalamalarının line chart üzerinde incelenmesi için The Simpsons, Doctor Who, Friends, Grey's Anatomy ve Supernatural dizilerinin sezon ortalamaları kullanılmıştır. Eğilimdeki yönü ölçmek için doğrusal eğim ve Spearman korelasyonu hesaplanmıştır.

Sonuçlar:

| Dizi | Sezon sayısı | Sezon başına eğim | Spearman rho | p-değeri |
|---|---:|---:|---:|---:|
| Doctor Who | 13 | -0.1142 | -0.7692 | 0.0021 |
| Friends | 10 | 0.0313 | 0.6727 | 0.0330 |
| Grey's Anatomy | 23 | -0.0514 | -0.6206 | 0.0016 |
| Supernatural | 15 | -0.0764 | -0.7464 | 0.0014 |
| The Simpsons | 38 | -0.0255 | -0.6877 | <0.000001 |

Doctor Who, Grey's Anatomy, Supernatural ve The Simpsons için puanların sezon ilerledikçe düşen anlamlı bir eğilimi vardır. Friends için ise pozitif bir eğilim bulunmuştur. Bu sonuçlar dizilere göre değişmekte; genelleştirilmiş bir nedensellik göstermemektedir.

Grafik: `figures/q1_seasonal_fatigue_lineplot.png`

Veri özeti: `data/processed/season_rating_summary.csv` ve `data/processed/season_trend_results.csv`

### Araştırma 2: Türlere göre puan dağılımı ve varyans

Temiz veri setinde birincil tür olarak iki tür bulunmuştur:

| Tür | Bölüm sayısı | Ortalama | Medyan | Medyanın %95 güven aralığı | Varyans |
|---|---:|---:|---:|---:|---:|
| Comedy | 1396 | 8.1366 | 8.2 | 8.1-8.2 | 0.4198 |
| Drama | 1170 | 7.8732 | 7.9 | 7.8-7.9 | 0.3923 |

Comedy bölümlerinin ortalama ve medyan puanı Drama bölümlerinden biraz yüksektir. Comedy puanlarının varyansı da biraz daha büyük olduğundan Comedy türünde puanların daha geniş dağıldığı görülmektedir.

Grafikler:

- `figures/q2_genre_rating_boxplot.png`
- `figures/q2_genre_rating_variance.png`

Veri özeti: `data/processed/genre_statistics.csv`

### Araştırma 3: Bölüm süresi ile puan ilişkisi

Temiz bölüm kayıtlarında bölüm süresi ile bölüm puanı arasındaki ilişki Pearson ve Spearman korelasyonlarıyla incelenmiştir.

- **Pearson r:** -0.1895
- **Pearson p-değeri:** 3.6256e-22
- **Spearman rho:** -0.1982
- **Spearman p-değeri:** 3.8300e-24
- Her iki yöntemde de ilişki **zayıf ve negatif** olarak sınıflandırılmıştır.

Korelasyonun istatistiksel olarak anlamlı olmasına rağmen bölüm süresindeki artışın puan üzerindeki etkisi güçlü değildir. Korelasyon nedensellik anlamına gelmez.

Grafik: `figures/q3_runtime_vs_rating_scatterplot.png`

Veri özeti: `data/processed/runtime_rating_correlations.csv`

## Sınırlılıklar

- Veriler TVMaze kullanıcı puanlarından oluşmaktadır ve izleyici kitlesinin tümünü temsil etmeyebilir.
- Dizi türü olarak yalnızca tür listesindeki birincil tür kullanılmıştır; çok türlü diziler tek bir kategoriyle temsil edilmiştir.
- Bazı dizilerin yayınlanmamış veya eksik bölüm kayıtları bulunabilir.
- TVMaze puanları zamana göre değişebilir ve analiz çekilen tarihteki API değerlerini göstermektedir.
- Sezon ortalamaları dizi başına puanlı bölüm sayılarından hesaplanmıştır; bazı erken veya geç sezonlarda kayıt sayısı daha az olabilir.
- Korelasyon ve trend analizleri gözlemsel sonuçlar olduğundan nedensel ilişki kurmaz.
- Aykırı değerler silinmemiş; raporda bayraklanarak tutulmuş ve eşleşen analizlerde grafikle birlikte incelenmiştir.

## Çalıştırma

### 1. Sanal ortam oluşturma ve paketleri kurma

```powershell
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
pip install scipy jupyter
```

### 2. Analiz notebook'unu açma

Nihai analiz dosyası:

```text
notebooks/tvmaze_analysis_improved.ipynb
```

Notebook'u JupyterLab veya VS Code ile açtıktan sonra tüm hücreleri çalıştırabilirsiniz:

- Jupyter: **Restart & Run All**
- VS Code: **Restart and Run All Cells**

Notebook, proje kökünü veya `notebooks/` klasörünü çalışma dizini olarak algılar; ham JSON, temiz CSV ve grafik dosyalarını ilgili `data/` ve `figures/` klasörlerine yeniden üretir.

### 3. Ham veriyi yeniden üretme

Ham veriyi tekrar indirmek için TVMaze API'ye internet erişimi gerekir. Notebook'un API toplama hücrelerini çalıştırarak `data/raw/tvmaze_raw.json` dosyası yeniden oluşturulabilir.

## Repository düzeni

```text
.
├── data/
│   ├── raw/
│   │   └── tvmaze_raw.json
│   └── processed/
│       ├── tvmaze_episodes_clean.csv
│       └── *_analiz_raporlari.csv
├── notebooks/
│   └── tvmaze_analysis_improved.ipynb
├── figures/
│   ├── q1_seasonal_fatigue_lineplot.png
│   ├── q2_genre_rating_boxplot.png
│   ├── q2_genre_rating_variance.png
│   └── q3_runtime_vs_rating_scatterplot.png
├── README.md
└── requirements.txt