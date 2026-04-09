# EDA Checklist voor Datajournalistiek

Printklare checklist en referentiegids voor verkenning van journalistieke datasets.

---

## 1. Volledige EDA-checklist

Volg deze stappen in volgorde. Markeer elke stap als je hem hebt uitgevoerd.

### Structuur en kwaliteit
- [ ] `df.shape` — hoeveel rijen en kolommen?
- [ ] `df.dtypes` — kloppen de kolomtypes? (getallen als object = probleem)
- [ ] `df.isnull().sum()` — hoeveel ontbrekende waarden per kolom?
- [ ] `df.duplicated().sum()` — zijn er dubbele rijen?
- [ ] `df.nunique()` — hoeveel unieke waarden per kolom? (onverwacht hoge/lage aantallen opsporen)
- [ ] `df.describe()` — beschrijvende statistieken voor numerieke kolommen

### Verdelingen
- [ ] Bereken gemiddelde én mediaan voor elke numerieke kolom
- [ ] Markeer kolommen waarbij gemiddelde en mediaan >20% afwijken (scheefheid / uitschieters)
- [ ] Bekijk histogrammen voor de belangrijkste variabelen

### Rankings
- [ ] Rangschik op elke metriek; toon top 5 en bottom 5
- [ ] Bereken afwijking van landelijk gemiddelde als percentage
- [ ] Verwerk per-capita of per-huishouden denominatoren waar beschikbaar

### Tijdtrends
- [ ] Bereken jaar-op-jaar verandering (absoluut én procentueel)
- [ ] Identificeer omslagpunten (waar keert de richting om?)
- [ ] Markeer langste ononderbroken stijging / daling

### Groepsvergelijkingen
- [ ] Groepeer op categorische variabelen (provincie, stedelijkheid, jaar)
- [ ] Bereken groepsgemiddelden én medianen
- [ ] Voeg afwijking van nationaal gemiddelde toe

### Uitschieters
- [ ] Gebruik IQR-methode: Q1 − 1,5×IQR en Q3 + 1,5×IQR als grenzen
- [ ] Formuleer elke uitschieter als potentiële verhaalhaak
- [ ] Controleer of uitschieters datakwaliteitsproblemen kunnen zijn (bijv. foute invoer)

### Correlaties
- [ ] Bereken Pearson-correlatiematrix voor alle numerieke kolommen
- [ ] Markeer sterke correlaties (|r| > 0,7)
- [ ] Waarschuw voor de ecologische fout bij geaggregeerde geografische data (zie sectie 3)
- [ ] Controleer op verrassende zwakke correlaties (verwachte verbanden die ontbreken)

### Ontbrekende waarden
- [ ] Controleer of ontbrekende waarden clusteren geografisch of temporeel
- [ ] Is het patroon van missingness zelf een verhaal?
- [ ] Onderscheid tussen "niet gemeten" en "niet van toepassing" en "geheim"

---

## 2. Statistisch testoverzicht — wanneer gebruik je wat?

| Situatie | Aanbevolen test | Wanneer te gebruiken |
|---------|----------------|---------------------|
| Vergelijk twee groepen (normaalverdeling) | **t-toets (ongepaird)** | Twee onafhankelijke groepen, metrische variabele, normaalverdeling |
| Vergelijk twee groepen (niet-normaal) | **Mann-Whitney U** | Niet-normaal verdeelde data of ordinale variabelen; robuust alternatief voor t-toets |
| Vergelijk meer dan twee groepen | **Kruskal-Wallis** | Meerdere onafhankelijke groepen; niet-parametrisch alternatief voor ANOVA |
| Verband tussen twee categorische variabelen | **Chi-kwadraat** | Kruistabellen; tel data per cel |
| Verband tussen twee continue variabelen | **Pearson r** | Lineair verband, beide normaalverdeeld |
| Verband bij niet-normaalverdeling | **Spearman ρ** | Monotoon verband zonder normaliteitsaanname; goed voor ranggegevens |
| Trend over tijd | **Sen's slope + Mann-Kendall** | Trenddetectie in tijdreeksen zonder normaliteitsaanname |

### Vuistregels

- Gebruik **noot-parametrische tests** (Mann-Whitney, Kruskal-Wallis, Spearman) tenzij je zeker weet dat de data normaalverdeeld is. In Nederlandse overheidsdata is dat zelden het geval — inkomen, vermogen, energieverbruik zijn doorgaans scheef verdeeld.
- Rapporteer altijd zowel de teststatistiek als de **effectgrootte** (niet alleen p-waarde). Een significant resultaat bij N=1000 gemeenten kan triviaal klein zijn.
- Gebruik **Bonferroni-correctie** als je meerdere tests tegelijk uitvoert op dezelfde dataset (deel het significantieniveau door het aantal tests).

### Python-code voor veelgebruikte tests

```python
from scipy import stats

# Mann-Whitney U (twee groepen vergelijken)
groep_a = df[df['categorie'] == 'A']['waarde']
groep_b = df[df['categorie'] == 'B']['waarde']
stat, p = stats.mannwhitneyu(groep_a, groep_b, alternative='two-sided')
print(f"Mann-Whitney U: stat={stat:.1f}, p={p:.4f}")

# Kruskal-Wallis (meerdere groepen)
groepen = [df[df['provincie'] == p]['waarde'] for p in df['provincie'].unique()]
stat, p = stats.kruskal(*groepen)
print(f"Kruskal-Wallis: stat={stat:.1f}, p={p:.4f}")

# Chi-kwadraat (categorische variabelen)
kruistabel = pd.crosstab(df['categorie_a'], df['categorie_b'])
stat, p, df_vrijheid, verwacht = stats.chi2_contingency(kruistabel)
print(f"Chi-kwadraat: stat={stat:.1f}, p={p:.4f}, df={df_vrijheid}")

# Spearman (niet-normaal verband)
r, p = stats.spearmanr(df['variabele_1'], df['variabele_2'])
print(f"Spearman ρ: {r:.3f}, p={p:.4f}")
```

---

## 3. De ecologische fout (ecological fallacy)

**Wat is het?** Een conclusie op individueel niveau trekken op basis van geaggregeerde geografische data.

**Voorbeeld**: Als gemeenten met een hogere gemiddeld inkomen ook meer zonnepanelen hebben, betekent dit *niet* dat rijke huishoudens meer zonnepanelen hebben. De correlatie op gemeenteniveau kan worden veroorzaakt door talloze andere factoren (type woningen, subsidieprogramma's, lokaal beleid, ruimtelijke indeling).

**Waarom is dit relevant voor datajournalistiek?** CBS-data is bijna altijd geaggregeerd op gemeente-, wijk- of buurtniveau. Correlaties op dat niveau zijn bruikbaar voor het vinden van patronen, maar je mag er **geen causale verbanden** of **individuele gedragspatronen** uit afleiden.

**Hoe te rapporteren?** Voeg altijd toe:

> "Deze analyse laat zien dat gemeenten met X doorgaans ook Y hebben. Of dit geldt voor individuele huishoudens, is met deze data niet te bepalen."

**Code: correlatie + waarschuwing automatisch toevoegen**

```python
def correleer_met_waarschuwing(df, kolom_1, kolom_2, niveau='gemeente'):
    """Bereken correlatie en voeg automatisch ecologische fout-waarschuwing toe."""
    import scipy.stats as stats
    
    geldig = df[[kolom_1, kolom_2]].dropna()
    r, p = stats.pearsonr(geldig[kolom_1], geldig[kolom_2])
    
    print(f"Pearson r({kolom_1} ↔ {kolom_2}) = {r:.3f}  (p = {p:.4f})")
    
    if abs(r) > 0.7:
        sterkte = "sterk"
    elif abs(r) > 0.4:
        sterkte = "matig"
    else:
        sterkte = "zwak"
    
    richting = "positief" if r > 0 else "negatief"
    print(f"Interpretatie: {sterkte} {richting} verband op {niveau}niveau")
    print(f"⚠️  Ecologische fout: dit verband op {niveau}niveau zegt niets over")
    print(f"    individuele huishoudens of personen (ecologische fout).")
    
    return r, p
```

---

## 4. Journalistieke denominatoren — veelgebruikte CBS-tabellen

Gebruik per-capita of per-huishouden denominatoren om grootteverschillen tussen gemeenten te corrigeren.

| Denominator | CBS-tabel | Opmerking |
|-------------|-----------|-----------|
| Bevolking per gemeente | `03759NED` | Jaarlijks bijgewerkt, per geslacht en leeftijdsgroep |
| Huishoudens per gemeente | `71486NED` | Onderscheid naar huishoudenstype |
| Woningvoorraad per gemeente | `82900NED` of opvolger | Let op: methodologie gewijzigd 2022 |
| Oppervlakte per gemeente (km²) | `70262NED` | Land- en wateroppervlak |
| Arbeidsplaatsen per gemeente | `80590NED` | Voor economische denominatoren |

```python
# Voorbeeld: per-capita berekening
from cbs_client import CBSClient  # Gebruik cbs-statline-skill om te downloaden

# Bevolkingstabel ophalen
client = CBSClient()
bevolking = client.get_data(
    '03759NED',
    filters="Perioden eq '2023JJ00' and Geslacht eq 'T001038'",
    select=['RegioS', 'BevolkingAanHetEindeVanDePeriode_5']
)
bevolking.columns = ['regio_code', 'bevolking']
bevolking['regio_code'] = bevolking['regio_code'].str.strip()

# Koppelen en per-capita berekenen
df_per_capita = df.merge(bevolking, on='regio_code', how='left')
df_per_capita['waarde_per_1000_inwoners'] = (
    df_per_capita['waarde'] / df_per_capita['bevolking'] * 1000
).round(2)
```

---

## 5. Nederlandse getalpresentatie voor publicatie

Gebruik de juiste notatie afhankelijk van de taal van het artikel.

| Situatie | Decimaalteken | Duizendtalteken | Voorbeeld |
|---------|---------------|-----------------|-----------|
| Nederlandstalig artikel | `,` (komma) | `.` (punt) | 1.234.567,8% |
| Engelstalig artikel | `.` (punt) | `,` (komma) | 1,234,567.8% |
| Code (Python/pandas) | `.` (punt) | geen of `_` | 1234567.8 |

### Formatteren voor publicatie

```python
def formatteer_nl(getal, decimalen=1, eenheid=''):
    """Formatteer getal voor Nederlandstalige publicatie."""
    if getal is None or pd.isna(getal):
        return 'n.b.'
    # Python's locale-onafhankelijke aanpak:
    tekst = f"{getal:,.{decimalen}f}"   # "1,234.5"
    nl = tekst.replace(',', 'X').replace('.', ',').replace('X', '.')  # "1.234,5"
    return f"{nl}{eenheid}"

# Gebruik in een findings-samenvatting:
print(f"Gemiddelde: {formatteer_nl(1234.5, decimalen=1)} warmtepompen per 1.000 woningen")
print(f"Groei: {formatteer_nl(34.2, decimalen=1)}%")
```

### Afrondingsconventies voor journalistiek

- Percentages: **1 decimaal** (tenzij het getal zelf al klein is, bijv. 0,03%)
- Grote absolute aantallen: afgerond op **duizendtallen** met ×1.000 of ×1.000.000 vermeld
- Geldbedragen: op **euro's** afgerond tenzij precies (kostprijs per eenheid)
- Groeipercentages: altijd met `+` of `−` prefix: +34,2% of −3,1%
- Kleine aantallen (< 100): geen decimalen tenzij écht relevant
