# Grid Up Datathon — Trafo Bazlı Elektrik Tüketimi Tahmini

Gdz ve Adm Elektrik Dağıtım A.Ş. tarafından düzenlenen Grid Up Datathon için
trafo bazlı günlük elektrik tüketimini tahmin eden bir makine öğrenmesi modeli.

## Görev
Geçmiş tüketim verisi + hava durumu / özel gün gibi ek verilerle,
gelecekteki (Nisan–Temmuz 2026) trafo bazlı tüketimi tahmin etmek.

## Metrik
RMSLE (Root Mean Squared Logarithmic Error) 


## Klasör Yapısı
- `data/raw/` — Kaggle'dan indirilen ham veri (train.csv, test.csv, sample_submission.csv)
- `data/processed/` — Temizlenmiş / feature-engineered veri
- `data/external/` — Dış kaynaklardan eklenen veri (hava durumu, özel günler vb.)
- `notebooks/` — EDA ve deneme notebook'ları
- `src/` — Yeniden kullanılabilir kod (feature engineering, model, utils)
- `models/` — Eğitilmiş model dosyaları
- `submissions/` — Kaggle'a gönderilen tahmin dosyaları
- `reports/figures/` — Görselleştirmeler (final sunumu için)
