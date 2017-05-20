# Poznámky - 2016/2017

*V tomto dokumentu jsem se pokusil sumarizovat informace, ručně vytažené ze slajdů přednášek. Píšu je formou "jak mi to padne na jazyk", takže to může být nespisovně, napůl anglicky, všelijak. Pull requesty vítány.*

## Crawling
- Automatický sběr obsahu na webu, rekurzivně se prochází linky na stránkách a zanořuje se hlouběji a hlouběji.
- Existují specializované nástroje - bots, spiders. Ty většinou umí aplikovat policies - nenavštěvovat tutéž stránku víckrát, ignorovat linky na obrázky, css, atd.
- Problémy:
    - Web je moc velký, musí se vybrat co stahovat.
    - Etika - nedosovat servery, používat User-Agenta, `robots.txt`.
- Terminologie:
    - Seed pages - seznam počátečních URL, ze kterých se pak jede dál
    - Frontier - seznam URL, které čekají na prozkoumání, tj. jakási fronta
    - Fetcher - řeší stahování stránky
    - Link Extractor - řeší parsování stránky a přidávání nových URL do Frontieru (to jsme dělali v úkolu pomocí např. CSS selektorů)
    - URL Filter, Duplicate elmiminator, URL prioritizer

#### Strategie crawlování
- BFS
    - klasický grafový BFS, čim dřív přidám URL do fronty, tím dřív ho pak resolvnu
- Backlink
    - před rozhodnutím, kterou stránku procesovat, si je seřadím podle toho, kolik URL v seznamu na ní ukazuje (tj. kolikrát je v seznamu)
    - Jako první budu řešit stránky, na který je hodně odkazů, takže asi budou důležitý.

Pro kontrolu, kdy se stránka naposled změnila a jestli má cenu jí přecrawlovávat můžu použít `HTTP HEAD`

Je třeba řešit URL normalizaci - transformace URL do její kanonické formy. Něco jako v Linuxu převádění relativní cesty, symlinky atd. na absolutní cestu. Tady se řeší např.:
- Velikost písmen
- Percent-encoding (%20 místo mezery)
- Resolving `.` a `..`

#### robots.txt
- Říká co můžu a co nemůžu indexovat (ale nikdo to neenforcuje - je to o etice)
- Příklad - zakázat GoogleBotu přístup do secret, ale všichni ostatní můžou:
```
User-agent: GoogleBot
Disallow: /secret/
Crawl-delay: 5

User-agent: *
Disallow:
```
- Tohle jde určovat i přímo v HTML stránky (*head.meta*).
- V headeru HTTP lze posílat direktivy botům: `noindex`, `noarchive`, `all`, `nofollow` ...

- Sitemap - standardizovaný XML formát, jak pomoct robotům v indexování:
```xml
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema‐instance"
        xsi:schemaLocation="http://www.sitemaps.org/schemas/sitemap/0.9 http://www.sitemaps.org/schemas/sitemap/
    <url>
        <loc>http://fit.cvut.cz/</loc>
        <lastmod>2014‐02‐27T15:26:35+00:00</lastmod>
        <changefreq>hourly</changefreq>
        <priority>1.0</priority>
    </url>
</urlset>
```

#### Specific content
- Deep Web - schovaný za formama, přihlašováním, paywally
    - Kontextuální (podle fyzické lokace)
    - Dynamické stránky (response na search query)
    - Non-HTML (videa, audio)
    - Soukromý web

- Obecně se dá těžko projít, dají se dělat věci jako hledání formů v HTML, zkoušet je vyplňovat, upravovat URL query ručně.
- Některý CAPTCHA jdou obejít, některý ne.
- AJAX stránky - jdou crawlovat když jsou dobře napsaný (např.  query `../#!inbox` bot předělá na `../_escaped_fragment_=inbox`, a tohle Google pochopí a naservíruje content, ale obecně problém.
- Spider pasti, dynamický nekonečný generování linků.

- Obsah stránky lze dolovat klasicky nástrojema pro zpracování textu, machine learning, regexpama (odhalí email adresy, telefonní čísla atp.), nějakým učicím automatem, který si postaví z HTML strom a hledá vhodný patterny.
- HTML může být taky anotovaný, aby boti věděli co to je. Např.
```html
<p vocab="http://schema.org/" typeof="Person">
    <span property="name">Christopher Froome</span> was sponsored by
    <span property="sponsor" typeof="http://schema.org/Organization">
    <a property="url" href="http://www.skysports.com/">Sky</a></span> in the Tour de France.
</p>
```
- Je třeba HTML fixnout, ne vždy bude validní.

## Web Data Mining - Indexing & Document retrieval
#### Information retrieval
- Nalezení dokumentů, který uživatelé chtěji, tedy:
    - Na základě sady existujících dokumentů a dotazu
    - Vytvoř ohodnocenou sadu relevantních dokumentů
    - Chci to rychle, napříč milionama nestrukturovaných dokumentů, spolehlivě, škálovatelně.

#### Modely
- jak jsou dokumenty reprezentované
- Boolean model
    - V dokumentech se uvažuje jenom to, jestli se v nich daný term vyskytuje nebo ne (0/1)
    - Queries se udávají booleovskými operátory a termy stylem `term1 AND (term2 OR term3)`.
    - Pro retrieval se řeší *jenom* přesná shoda - dokument buď relevantní je, nebo není.
- Vector Space model
    - Dobře známý a obecně používaný model
    - Používá **TF-IDF** váhové schéma:
        - TF - term frequency - počet výskytů termu v dokumentů (normalizováno celkovým počtem termů v dokumentu)
        - IDF - inverse document frequency - udává, jak je slovo běžné napříč všemi dokumenty: ![tfidf formula](resources/tfidf.png)
        - *TF-IDF = TF × IDF*

    - Příklad:

        > Slovo se vyskytuje 5x v dokumentu se 100 unikátními slovy -> TF = 5/100

        > Máme 10 000 dokumentů a slovo je (alespoň jednou) ve 100 z nich -> IDF = log(10000/100) = 2

        > TF-IDF = 5/100 * 2 = 1/10

#### Vector space ranking
- který dokument zvolit, který je nejpodobnější?

- Query je množina slov, na jejím základě nějak vyberu dokumenty, které obsahují alespoň jedno ze slov
- Query reprezentuju jako vektor (slova -> čísla) (_"hello world hello" -> (1, 0.5)_, první složka je pro _hello_, druhá pro _world_, normalizovaně)
- Každý kandidátní dokument (docX) reprezentuju jako vektor, kde složky vektoru jsou TF-IDF slov z **query**: (tfidf(hello, docX), tfidf(world, docX))
- Spočtu vzdálenost vektoru Query a vektoru *každého* z dokumentů, seřadím od nejbližšího, to jsou moje výsledky.

- Vzdálenosti:
    - Eukleidovská - klasicky odmocnina ze součtu druhých mocnin, není dobrá, protože když budu mít v query (term1, term2), tak potom s každou další dvojicí termů term1 a term2 v dokumentu bude vzdálenost růst, i když by logicky měla bejt cca stejná.
    - Cosinová - lepší, místo vzdálenosti vektorů se počítá úhel mezi nimi, takže předchozí problém nenastává:

        ![](resources/cosine-distance.PNG)

        Ve jmenovateli je jednoduše násobení vektorů po složkách.
        Pokud jsou vektory totožné, mají úhel 0 a cos(0) = 1.

#### Evaluation measures
- jak hodnotit kvalitu systému?
- Precision - kolik z vrácených dokumentů je skutečně relevantních? - P(relevant | retrieved)
- Recall - kolik z celkem relevantních dokumentů bylo vráceno? - P(retrieved | relevant)
- F-measure - tradeoff mezi precision/recall:

    ![](resources/fmeasure.PNG)

#### Indexing
- sémantika dokumentů a queries jde zachytit množinou index termů - klíčových slov
- **Document index** - klíčová slova, popisující dokument
- indexing - proces vytváření indexu pro každý dokument
- index dokumentů pracuje s *vocabulary* - množinou klíčových slov
    - manuální indexování
        - omezená vocabulary, vytvořena před indexováním
        - kvalitní keywordy, člověk rozumí obsahu, abstrahuje
        - člověk si nevzpomene na celou knihu při indexaci, drahé, pomalé, nekonzistentní
    - automatické indexování
        - nekontrolovaná vocabulary, tvoří se dynamicky při indexaci
        - chcem imitovat lidi, používáme omezený počet termů

- **Inverted index**
    - mapování "keyword" -> "doc1, doc2"
    - tohle je většinou to, co se myslí indexem
    - buduje se předem, na ve chvíli kdy se řeší query
    - používají se standardní struktury pro mapy - hashtables, B-trees
        - hashtable rychlejší lookup, ale při rozumně omezeným počtu termů
        - B-tree delší hledání, ale umí prefix search a najít podobný slova

## Web Data Mining - Text Mining
- Extrakce informací a znalostí z textu

- Proces text miningu:
    1. Sběr dokumentů
    2. Preprocessing dokumentů
    3. Transformace textu, generování featur
    4. Redukce dimenzionality - výběr featur
    5. Pattern discovery, klasicky data mining
    6. Interpretace výsledků

- Tokenizace
    - Nalezení vět, slov -- seskupování znaků do logických celků

- Snížení dimenzionality, převedení tokenů do standardní podoby
    - Lemmatizace - převod do základního tvaru (was, is, are -> be)
    - Stemming - určení kořenu slova (waiting -> wait), částečně řízen pravidly, částečně slovníkově
    - Odstranění stopwords (a, the, it, they)
    - Synonyma, antonyma

- Sentence splitting
    - Rozsekání textu na věty
    - Není triviální, tečka ne vždy ukončuje větu

- Part-of-speech tagging
    - Rozpoznání slovních druhů (noun, proper noun, adjective, verb, pronoun...)
    - Stejná slova můžou mít samozřejmě různé výrazy, není to easy
    - Postupy na tagování:
        - Už mám otagovaný nějaký set dokumentů, tím naučím nějaký klasifikátor
        - Regexp pravidla
        - Obecně pravidla (pokud mam "to XXX", pak je XXX sloveso)
        - N-gramy

- Named entity recognition
    - Identifikace entit a klasifikace, co to je za entitu
    - NER dělá:
        - Rozpoznání entit
        - Klasifikace
        - Disambiguation tím, že ke třídě ještě přiřadím třeba Wikipedia URL s tou entitou
    - Založeno na gramatikách nebo Gazetteers, což jsou seznamy/slovníky států, lidí, geolokací
    - Množina tříd může být předdefinovaná, nebo dynamicky tvořená (LOC, GPE, ORG, MISC)
    - Slova jsou mnohoznačná, NER může použít *linking*, aby se tomu vyhnul - k entitě přidá odkaz na nějakej jednoznačnej zdroj.
    - *Coreference* - různý kusy textu označují tutéž entitu, chcem je tak i stejně označit (Mary Smith, Mrs. Smith)

- Relation Extraction
    - Vytěžování sémantických vztahů mezi entitami (X pracuje pro Y)
    - Naivně se dají hledat fixní slovní patterny, ale to neškáluje
    - Řeší se lingvistickou analýzou (tagging slovních druhů, sestaví se parsing tree, z toho se určí podmět, předmět, přísudek)

- Opinion Mining
    - Identifikace názorů, emocí, sentimentu z textu - většinou se zkoumá pozitivní, negativní nebo neutrální

    1. Opinion Extraction - identifikace textu, který obsahuje názor, emoce.
    2. Sentiment Analysis - neřeší komplexnější názory, jen hledá pozitivní/negativní.
    3. Opinion Summarization - vypočtení celkového názoru z předchozího bodu

    - Lexicon-based - mám seznam dobrých a špatných slov a proti tomu porovnávám. Jsou tu ale issues, třeba před tím slovem dám *not*, nebo je to jenom otázka, nemá to sentiment. Můžu dál expandovat lexikon například hledáním synonym (stejná polarita), antonym (opačná polarita) - WordNet.

    - Sentiment learning
        - Supervised - mám anotovaná data, na kterých trénuju, například to seberu z recenzí produktů a hvězdiček
        - Unsupervised - sentiment bude na daných frázích slovních druhů, např "ADJ NN"

- Text summarization
    - Cíl je vytvoření abstraktu, shrnutí textu
    - Kroky:
        - Content selection - jaký věty budu vůbec ve zdroji uvažovat
        - Information ordering - jak informace uspořádám
        - Sentence realization - "vyčištění" vět, zjednodušení
    - Postupy:
        - Frequency-based - hledám často se opakující N-gramy, ty potom nějakým způsobem spojím.
        - Baseline Single Document - hledám nejdůležitější slova v textu, k nim pak clustery okolních důležitých slov, pak vypočtu skóre každého z clusterů a ve finále budu uvažovat věty, které mají clustery s nejvyššími skóre.

- *NIF*
    - Natural language processing Interchange Format
    - Standard anotací, má URI schéma

## Web Data Mining - Social Network Analysis
- Je to o dost starší než Facebook, studuje lidské vztahy pomocí teorie grafů
- Typy analýzy:
    - Sociocentrická - analýza celé sítě, identifikace globálních struktur
    - Egocentrická - analýza okolí jednoho člověka, kvantifikace interakcí mezi jedním uzlem a ostatními
    - Knowledge-based - pochází z computer science, kvantifikuje interakce mezi uživateli, skupinami a dalšími entitami v síti

- Kevin Bacon / Paul Erdös number - six degrees of separation
- Measures of centrality:
    - Metriky pro měření důležitosti, popularity nebo sociálního kapitálu uzlu v síti, na základě jejich connection patterns
    - Degree centrality - stupeň uzlu - počet hran vedoucí do/z uzlu
    - Closeness - *1 / (součet vzdáleností do všech ostatních uzlů)*
    - Betweenness - počet nejkratších cest všech ostatních uzlů, které vedou skrz uzel X dělený počtem všech nejkratších cest
    - Eigenvector - podobně jako PageRank, iterativně. Uzel s vysokou Eigenvector centralitou je propojen s uzly, které ji mají taky velkou, tedy "kdo je propojen s hodně propojenými uzly?"

#### Clustering
- Clustering coefficient
    - pravděpodobnost, že dva náhodní kamarádi daného uzlu jsou taky navzájem kamarádi. Udává koncentrovanost vztahů v clusteru, či globálně úroveň clusterovatelnosti.
    - (*počet hran mezi sousedy uzlu X*) děleno (*maximální počet hran mezi sousedy uzlu X*)
    - Viz níže - šedé i černé hrany jsou platné, červené neplatné

    ![](resources/clustering-coefficient.PNG)

- Bridge
    - Smazáním hrany mezi dvěma uzly by se tyto uzly ocitly ve dvou různých komponentách (vzácné)
    - **Local bridge** - pokud dva uzly nemají žádné společné přátele, tedy rozpojením by jejich vzdálenost vzrostla na víc než dva.
    - Embeddedness(A,B) - počet společných sousedů A, B

#### Network-level characteristics
- **Density**
    - Poměr hran grafu vůči všem možným hranám (pokud by byl plný, tj. neorientovaný *n(n-1)/2*)

- **Reciprocity**
    - U orientovaných grafů
    - Poměr spojení, které vede oběma směry, vůči všem spojením.

- **Components**
    - Klasická komponenta grafu - spojitý podgraf.

- **Homophily**
    - Mám dva typy uzlů, např. muži-ženy, bílý-černý. Homophily je, pokud se spolu druží spíš v rámci sexu/rasy než aby se družili se všema.
    - Pokud je počet cross-gender uzlů výrazně nižší než 2*p*q (kde p, q je podíl té které skupiny), pak je homophilie.

- **Community detection**
    - Jsou CS způsoby a sociologický způsoby
    - Obecně, komunita je podmnožina uzlů, ze které do zbytku grafu vede míň hran než kolika je prolinkovaná komunita (ještě neexistuje přesná definice)

    - **Cliques**
        - Úplný podgraf - každý je s každým v rámci kliky propojen

    - **Partitioning**
        - Rozdělení grafu na dva clustery, chceme minimalizovat počet uřízlých hran
        - *Kernighan-Lin* algoritmus:
            *TODO? Nevim jestli to má význam.*
        - *Hierarchical clustering*
            - Sestavení stromu clusterování na základě nějaké podobnosti
        - *Betweenness* - spočtu betweenness všech dvojic uzlů, uříznu hrany s nejmenší hodnotou
        - *Island method* - pro každý uzel spočtu "výšku", například centralitu, a pak postupně zvedám "vodu" - nějaký threshold, který mi graf rozdělí.
        - Komunity se ale často překrývají, existují na to taky nějaký algoritmy, jak je identifikovat.

- **Link prediction**
    - Sítě jsou vysoce dynamické, pořád se mění. Chci predikovat, kde se budou tvořit hrany (doporučování přátel, interakce mezi proteiny v bioinformatice, doporučovací systémy)
    - Využívá se známých charakteristik:
        - Svět je malý, průměrná vzdálenost v síti je docela malá v porovnání s velikostí sítě
        - Většina nodů má krátký linky
        - Existují clustery přátel, známých a tak podobně
    - Obecně těžký problém, mám 1/(V^2) pravděpodobnost, že se trefím.
    - Algoritmy:
        - Graph Distance - čím kratší cesta, tím spíš se vytvoří hrana. *score(x,y) = -shortestPath(x,y)*
        - Common Neighbors - *score(x,y) = <počet společných přátel>*

- **Multinode networks**
    - Grafy, kde je víc typů uzlů, např. lidi a společnosti(linkedin), přátelé a stránky (fb)
    - *Affiliation networks* - nepřímé, tranzitivní propojení přes druhý typ uzlu. Počet hran se sečte a ve výsledném grafu tím bude ohodnocena odpovídající hrana:

        ![](resources/affiliation-network.PNG)