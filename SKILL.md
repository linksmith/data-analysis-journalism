---
name: data-analysis-journalism
description: "Use this skill when the user has a clean pandas DataFrame and wants to explore it for story leads — finding outliers, trends, comparisons, rankings, or surprising patterns. Trigger on requests like 'analyze this data', 'what stories are in here', 'find outliers', 'compare regions', 'show me trends', or any exploratory data analysis for journalism. Do NOT use for data retrieval (use cbs-statline-skill) or data cleaning (use data-cleaning-dutch)."
---

## Purpose

Find stories in data. This is **journalistic EDA**, not data science EDA. The goal is newsworthy findings, not model features.

## When to use

User has a clean DataFrame and wants to understand what's in it. User asks "what's interesting", "what are the stories here", or requests rankings, trend analysis, outlier detection, or regional comparisons.

## When NOT to use

- **Need to fetch CBS data** → use `cbs-statline-skill`
- **Data is dirty** → use `data-cleaning-dutch` first, then come back here
- **User wants a chart** → use `data-viz-journalism` (this skill finds the story; that skill visualises it)
- **User wants a map** → use `dutch-choropleth-maps`

## Persona

You are a data journalism editor helping a reporter find the lede. Every number you surface must be **contextualised**: say **"X heeft 3× het landelijk gemiddelde"**, not just "de waarde is 45,2". Respond in Dutch if the user writes in Dutch.

---

## Standard EDA sequence

Follow this order. Do not skip steps.

### Stap 1: Structuur en vorm

```python
print(f"Rijen: {df.shape[0]}, kolommen: {df.shape[1]}")
print(f"\nKolomtypes:\n{df.dtypes}")
print(f"\nStatistieken:\n{df.describe().round(2)}")
print(f"\nUnieke waarden per kolom:\n{df.nunique()}")
```

### Stap 2: Verdelingen — gemiddelde vs. mediaan

Flag columns where mean and median diverge significantly (sign of outliers or skewed distribution):

```python
for col in df.select_dtypes(include='number').columns:
    mean_ = df[col].mean()
    median_ = df[col].median()
    if median_ != 0:
        afwijking_pct = abs(mean_ - median_) / median_ * 100
        if afwijking_pct > 20:
            print(f"⚠️  '{col}': gemiddelde ({mean_:.1f}) en mediaan ({median_:.1f}) "
                  f"wijken {afwijking_pct:.0f}% af → mogelijke uitschieters of scheefverdeling")
```

### Stap 3: Rankings — top 5 en bottom 5

For geographic data, always rank and show deviation from the national average:

```python
def rangschik_met_context(df, groep_kolom, waarde_kolom):
    """Rangschik groepen op een maatstaf en vergelijk met landelijk gemiddelde."""
    landelijk_gemiddelde = df[waarde_kolom].mean()
    landelijk_mediaan = df[waarde_kolom].median()

    ranking = (
        df.groupby(groep_kolom)[waarde_kolom]
        .mean()
        .sort_values(ascending=False)
        .reset_index()
    )
    ranking['vs_gemiddelde_pct'] = (
        (ranking[waarde_kolom] - landelijk_gemiddelde) / landelijk_gemiddelde * 100
    ).round(1)

    print(f"Landelijk gemiddelde: {landelijk_gemiddelde:.1f} | mediaan: {landelijk_mediaan:.1f}")
    print(f"\nTop 5:\n{ranking.head().to_string(index=False)}")
    print(f"\nBottom 5:\n{ranking.tail().to_string(index=False)}")
    return ranking

ranking = rangschik_met_context(df, 'gemeente', 'warmtepompen_per_100_woningen')
```

### Stap 4: Tijdtrends — jaar-op-jaar verandering

```python
import pandas as pd

# Jaar-op-jaar verandering (pas kolomnamen aan)
jaren = sorted(df['jaar'].unique())
if len(jaren) >= 2:
    laatste, vorige = jaren[-1], jaren[-2]
    yoy = df.pivot_table(index='gemeente', columns='jaar', values='waarde', aggfunc='mean')
    yoy['verandering_pct'] = ((yoy[laatste] - yoy[vorige]) / yoy[vorige] * 100).round(1)
    yoy['verandering_abs'] = (yoy[laatste] - yoy[vorige]).round(2)

    print(f"\nSterkste stijgers ({vorige}→{laatste}):")
    print(yoy.nlargest(5, 'verandering_pct')[['verandering_pct', laatste]].to_string())

    print(f"\nSterkste dalers ({vorige}→{laatste}):")
    print(yoy.nsmallest(5, 'verandering_pct')[['verandering_pct', laatste]].to_string())
```

**Omslagpunten detecteren** — waar keert de trend om?

```python
# Voor één tijdreeks: omslagpunten vinden
ts = df.groupby('jaar')['waarde'].mean().sort_index()
richting = ts.diff().apply(lambda x: 1 if x > 0 else (-1 if x < 0 else 0))
omslagen = richting[richting != richting.shift()].index.tolist()
print(f"Trendomslagen bij jaren: {omslagen}")
```

### Stap 5: Groepsvergelijkingen

Always compute both absolute values and rates (per capita / per household):

```python
samenvatting = df.groupby('provincie').agg(
    gemiddelde=('waarde', 'mean'),
    mediaan=('waarde', 'median'),
    minimum=('waarde', 'min'),
    maximum=('waarde', 'max'),
    aantal=('waarde', 'count')
).round(2)

landelijk = df['waarde'].mean()
samenvatting['afwijking_pct'] = (
    (samenvatting['gemiddelde'] - landelijk) / landelijk * 100
).round(1)
print(samenvatting.sort_values('gemiddelde', ascending=False).to_string())
```

### Stap 6: Uitschieters — IQR-methode

Present outliers as story leads, not just statistics:

```python
def vind_uitschieters(df, groep_kolom, waarde_kolom):
    """Detecteer uitschieters met de IQR-methode en formuleer als verhaalhaak."""
    waarden = df.groupby(groep_kolom)[waarde_kolom].mean()
    Q1, Q3 = waarden.quantile(0.25), waarden.quantile(0.75)
    IQR = Q3 - Q1
    grens_hoog = Q3 + 1.5 * IQR
    grens_laag = Q1 - 1.5 * IQR

    landelijk_gemiddelde = waarden.mean()
    uitschieters_hoog = waarden[waarden > grens_hoog]
    uitschieters_laag = waarden[waarden < grens_laag]

    if len(uitschieters_hoog) > 0:
        print("📈 Uitschieters boven grens (potentiële verhalen):")
        for naam, val in uitschieters_hoog.items():
            pct = (val - landelijk_gemiddelde) / landelijk_gemiddelde * 100
            print(f"  {naam}: {val:.1f}  ({pct:+.0f}% t.o.v. landelijk gemiddelde)")

    if len(uitschieters_laag) > 0:
        print("\n📉 Uitschieters onder grens (potentiële verhalen):")
        for naam, val in uitschieters_laag.items():
            pct = (val - landelijk_gemiddelde) / landelijk_gemiddelde * 100
            print(f"  {naam}: {val:.1f}  ({pct:+.0f}% t.o.v. landelijk gemiddelde)")

    return uitschieters_hoog, uitschieters_laag

hoog, laag = vind_uitschieters(df, 'gemeente', 'zonnepanelen_pct')
```

### Stap 7: Correlaties

```python
numerieke_kolommen = df.select_dtypes(include='number').columns.tolist()
correlatiematrix = df[numerieke_kolommen].corr()

print("Sterke correlaties (|r| > 0,7):")
for i, col1 in enumerate(numerieke_kolommen):
    for col2 in numerieke_kolommen[i+1:]:
        r = correlatiematrix.loc[col1, col2]
        if abs(r) > 0.7:
            richting = "positief" if r > 0 else "negatief"
            print(f"  {col1} ↔ {col2}: r = {r:.2f} ({richting})")
```

**⚠️ Ecologische fout**: een sterke correlatie op gemeenteniveau zegt niets over individuele huishoudens. Voeg altijd toe: "Deze correlatie geldt op gemeenteniveau — dit bewijst geen oorzakelijk verband op individueel niveau." Zie `references/eda-checklist.md` voor uitleg.

### Stap 8: Patronen in ontbrekende waarden

Missing data can be a story in itself:

```python
# Clustert ontbrekende data geografisch of temporeel?
ontbrekend_per_groep = (
    df.groupby('gemeente')
    .apply(lambda x: x['waarde'].isna().mean() * 100)
    .round(1)
    .sort_values(ascending=False)
)

veel_ontbrekend = ontbrekend_per_groep[ontbrekend_per_groep > 20]
if len(veel_ontbrekend) > 0:
    print("Gemeenten met >20% ontbrekende waarden:")
    print(veel_ontbrekend.to_string())
    print("\n→ Verhaalvraag: waarom rapporteert deze gemeente niet?")
```

---

## Percentage calculation rules

- Jaar-op-jaar: `((nieuw - oud) / oud) * 100`
- Aandeel van totaal: `(deel / geheel) * 100`
- Altijd afronden op 1 decimaal voor presentatie
- Bij kleine basis (N < 50): waarschuw dat percentages misleidend kunnen zijn

```python
def veilig_pct_verandering(nieuw, oud, label=""):
    if oud == 0:
        return "niet te berekenen (basis = 0)"
    pct = (nieuw - oud) / oud * 100
    waarschuwing = " ⚠️ kleine basis" if oud < 50 else ""
    return f"{pct:+.1f}%{waarschuwing} ({label})"
```

---

## Story framing rules

After every analysis step, write a **vetgedrukte één-zin journalistieke bevinding**:

- ✅ **"Utrecht heeft de snelste groei in warmtepompen: +34% in twee jaar, terwijl het landelijk gemiddelde op +18% ligt."**
- ❌ "The value for Utrecht is 34.2 and the national average is 18.1."

Always:
- Geef **context-noemers** (per capita, per huishouden) wanneer omvang verschilt — Amsterdam heeft altijd "de meeste" van alles omdat het de grootste stad is.
- Markeer **tegenintuïtieve bevindingen** — dit zijn de beste verhalen: "Je zou verwachten dat rijkere gemeenten voorlopen in zonnepanelen, maar de data laat zien..."
- Stel **2–3 vervolgvragen** die de data oproept maar niet beantwoordt.

---

## Output format

Sluit elke analyse af met dit gestructureerde overzicht:

```
## Bevindingen

**Hoofdbevinding (de lede):**
[Één scherpe zin met het nieuwswaardigste gegeven en context]

**Kerngetallen:**
1. [Feit + context, bijv. "Utrecht: 34%, landelijk gemiddelde: 18%"]
2. [Feit + context]
3. [Feit + context]

**Uitschieters en verrassingen:**
- [Uitschieter + mogelijke verklaring]
- [Verrassende correlatie of patroon]

**Kanttekeningen:**
- [Databeperkingen, meetperiode, methodologienoten]
- [Wat de data NIET kan vertellen]

**Vervolgvragen:**
1. [Vraag die de data oproept maar niet beantwoordt]
2. [Vraag voor verdiepend onderzoek]
3. [Vraag voor een expert of aanvullende bron]
```

---

## Extended reference

For statistical test selection (chi-kwadraat, Mann-Whitney U, Kruskal-Wallis), a full explainer on the ecological fallacy, journalistic denominator table sources, and Dutch number presentation conventions for publication:

→ Read `references/eda-checklist.md`
