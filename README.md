# Schemas-Mutatielevering
Deze github repo bevat de XML-schema's voor mutatielevering.
XML-schema's worden gebruikt voor de mutatieleveringen BRK Kadastrale kaart en de BGT. Voor de Kadastrale kaart is een beschrijving van [Mutatieformaat Kadastrale kaart v.1.0](https://developer.kadaster.nl//schemas/brkkadastralekaart/v20190501/Mutatieformaat%20Kadastrale%20kaart%20v.1.0.pdf) op [developer.kadaster.nl](https://developer.kadaster.nl/schemas) te vinden. Voor de BGT is staat deze beschrijving op [pdok.nl](https://www.pdok.nl/bgt-mutatie). 
Ook is er een voorbeeld client implementatie beschikbaar op [github](https://github.com/PDOK/delta-download-ref-impl). 

# Ontwerp XML-mutatieformaat
Het XML-mutatieformaat dat door PDOK wordt geleverd, heeft als doel externe partijen te laten “meemuteren” met de actuele stand zoals die wordt bijgehouden in de bron van waaruit wordt geleverd. In het geval van de basisregistraties is dat de betreffende Landelijke Voorziening.
Bij het specificeren van dit mutatieformaat heeft PDOK enerzijds gestreefd naar een formaat dat de afnemer in staat stelt een 100% getrouwe kopie van de bron op te bouwen en anderzijds naar een formaat dat eenvoudig en flexibel te implementeren en snel te verwerken is. Dit document beschrijft het actuele concept-mutatieformaat dat hieruit is voortgekomen.

# Concepten
Een object in de werkelijkheid kan in de loop van de tijd wijzigen. Denk aan een pand dat wordt gebouwd, later door een verbouwing wordt uitgebreid en uiteindelijk weer gesloopt. De periode tussen twee van zulke wijzigingen noemen we hier een versie. Van elk object houdt de bron van waaruit geleverd wordt wijzigingshistorie bij, die bestaat uit op elkaar aansluitende versies met elk een begin- en een eindtijdstip. Hoe de werkelijkheid precies wordt afgebeeld op objecten en wanneer versies ontstaan en beëindigd worden, wordt door de bronhouder bepaald en verschilt dus mogelijk per bron.
Het hier uitgewerkte mutatieformaat introduceert een generiek mechanisme dat van deze verschillen abstraheert, zodat kennis van het historiemodel van de verschillende bronnen niet nodig is om de mutatieverwerking te implementeren en een lokale kopie van een bron op te bouwen. Om deze kopie te interpreteren, bijvoorbeeld de vraag te beantwoorden welke versie van een object de actuele toestand in de werkelijkheid weergeeft, is deze kennis echter wel vereist.
Uit het feit dat de bronhouder bepaalt wanneer versies ontstaan en beëindigd worden, volgt het belangrijkste beginsel van dit mutatieformaat: mutaties vinden niet plaats op het niveau van objecten, maar er worden individuele versies daarvan toegevoegd, gewijzigd of verwijderd. Immers is het niet mogelijk om mutaties op objectniveau door te voeren zonder het historiemodel van de betreffende bron te kennen en opnieuw te implementeren in elke applicatie die deze mutaties verwerkt. De volgende voorbeelden geven aan hoe een aantal typische gevallen omgezet worden naar mutaties in dit op versies gebaseerde model.

# Voorbeelden
 Object A ontstaat op 1 april:

>BAG, BGT, BRK: Toevoegen object A versie 1 met begindatum 1 april en einddatum leeg (omdat dit de momenteel geldige versie is).

Object A wijzigt op 1 mei:

>BAG, BGT, BRK: Wijzigen object A versie 1 om einddatum te veranderen van leeg naar 1 mei. Toevoegen object A versie 2 met begindatum 1 mei en einddatum leeg (omdat dit de momenteel geldige versie is).

Object A komt op 1 juni te vervallen:

>BGT, BRK: Wijzigen object A versie 2 om einddatum te veranderen van leeg naar 1 juni.
BAG: Als BGT, BRK, maar daarnaast: toevoegen object A versie 3 met begindatum 1 juni, einddatum leeg en status “beëindigd”.

Object A versie 1 blijkt een fout te bevatten, die hersteld moet worden zonder versie 2 aan te passen:

>BAG, BGT, BRK: Wijzigen object A versie 1 om de getroffen velden te veranderen van de foutieve in de juiste waarde.

Door een technische fout blijkt ten onrechte een versie 3 van object A geïntroduceerd te zijn (Let op het formaat is hier op voorbereid, BGT en BRK leveren echter geen herstel mutaties):

>BGT, BRK: Verwijderen object A versie 3.
BAG: Wijzigen object A versie 3 om het veld “inactief” op “ja” te zetten.

# Mutatiemodel
Als uitwerking van het mutatieformaat is gekozen voor het WAS-WORDT-model, omdat alle mogelijke mutaties op versies hierin intuïtief uitgedrukt kunnen worden. Wordt een versie toegevoegd, dan bestaat de mutatie uit enkel een WORDT. Wordt een versie gewijzigd, dan bestaat de mutatie uit een WAS en een WORDT die de toestand voor en na de wijziging aangeven. Wordt een versie definitief verwijderd, dan bestaat de mutatie enkel uit een WAS.
Aan een zuiver WAS-WORDT-model kleeft echter het bezwaar dat voor het terugvinden van de WAS in de kopie ofwel alle velden vergeleken moeten worden, ofwel bronspecifieke kennis nodig is om te weten welke velden voldoende zijn om een versie uniek te identificeren. Om hieraan tegemoet te komen, kent het hier beschreven mutatieformaat aan elke WORDT – bovenop de bronspecifieke velden – een uniek identificatienummer toe. Een latere WAS verwijst hiernaar terug, zodat enkel dit identificatienummer nodig is om de WAS in de kopie te vinden. Verdere velden vergelijken is niet nodig. Enkele bijkomende voordelen van het gebruik van een identificatienummer zijn:
-	Elke wijziging aan een versie is uniek aan te duiden en daarmee te traceren, omdat elke WORDT een nieuw identificatienummer krijgt. Dit geldt zelfs als een wijziging later exact ongedaan gemaakt wordt.
-	De afnemer kan ervoor kiezen zijn lokale kopie in een ander formaat op te slaan dan de bron gebruikt, bijvoorbeeld als hij niet alle velden nodig heeft. In een zuiver WAS-WORDT-model is dit niet mogelijk als daardoor de WAS niet meer met zekerheid te vergelijken is. Als echter het identificatienummer wordt opgeslagen, staat het de afnemer vrij de andere velden naar eigen goeddunken op te slaan.
Afnemers die geen aanvullend identificatienummer wensen te gebruiken, hebben de mogelijkheid dit te negeren en de WAS op basis van de datavelden te vinden (de zuivere WAS-WORDT-methode). Deze afnemers hebben dan de keuze of zij alle velden vergelijken, wat voor elke bron werkt, of een uniek identificerende subset van de velden gebruiken, die per bron verschilt. Tot slot is het mogelijk de WAS wel te vinden op basis van het aanvullende identificatienummer, maar vervolgens alsnog de datavelden te vergelijken om te valideren of de lokale kopie zich in de verwachte toestand bevindt.
Welke implementatie de afnemer ook kiest, dient wel benadrukt te worden dat het identificatienummer een implementatiehulp is en geen gegeven uit de oorspronkelijke bron. Als zodanig dient de afnemer het niet naar buiten toe beschikbaar te stellen of deel van een interface te maken.

# Technische opbouw
Per batch van een aantal duizend mutaties (WORDT, WAS-WORDT of WAS) ontstaat een XML-bestand, waarin mutaties voor verschillende objecttypen (bijv. pand of wegdeel) door elkaar heen kunnen voorkomen.
Als een versie meermaals binnen één bestand gemuteerd wordt, dan staan deze mutaties in de juiste volgorde in het bestand. Met andere woorden: als het bestand van boven naar beneden verwerkt wordt, kan het nooit zo zijn dat een WAS verwijst naar een toestand die pas verderop in het bestand ontstaat d.m.v. een WORDT. Sorteren in de verwerkende applicatie is dus niet nodig.
Het totaal van XML-bestanden van de opgevraagde mutatie wordt gezipt, waarbij zowel de alfabetische volgorde van de bestandsnamen als de fysieke volgorde van de bestanden in de zip overeenkomt met de volgorde waarin de bestanden verwerkt moeten worden. Dit maakt het mogelijk de zip op “streaming” wijze te verwerken, oftewel met de verwerking te starten terwijl de zip nog gedownload en gedecomprimeerd wordt.
Inhoudelijk zijn de individuele mutatiebestanden als volgt opgebouwd:
-	De kop van het bestand bevat (nader te specificeren) metadatavelden die aangeven bij welke opgevraagde mutatieset het XML-bestand hoort.
-	Eén of meer elementen van het type mutatieGroep. Een mutatiegroep omvat mutaties die logisch gezien bij elkaar horen en in één keer verwerkt moet worden om een consistente toestand te behouden.
o	Eén of meer elementen van het type mutatie. De attributen objectType en objectId geven aan waarop de mutatie betrekking heeft.
	Eén element van het type wordt. Elke wordt heeft een attribuut id dat de toestand van de versie na deze mutatie uniek identificeert.
	OF één element van het type was. Elke was heeft een attribuut id dat terugverwijst naar het id van een eerdere wordt. Dit is de toestand die komt te vervallen.
	OF één was en één wordt. Beiden hebben een attribuut id, respectievelijk van de vervallen en van de nieuwe toestand.
Binnen de elementen was en wordt wordt gebruikgemaakt van het oorspronkelijke XML-formaat van de betreffende bron. In het geval van de BGT staat hier bijvoorbeeld een CityGML-object.

# Initiële Stand
De initiële stand van een dataset wordt aangeboden in het zelfde formaat als bij een wijziging, waarbij alle objecten in WORDT formaat worden aangeboden. Wijzigingen worden 31 dagen gehouden en als een klant langer geen update doet dan moet een nieuwe initiële stand downloaden.

# Interessegebieden
Afnemers van een interessegebied krijgen:
 
1.	de mogelijkheid om mutaties af te nemen van een door henzelf te bepalen interessegebied
2.	de mogelijkheid om de precieze afbakening (= coördinaten) van het interessegebied op te vragen zodat men deze kan visualiseren of op andere wijze kan gebruiken om misverstanden te voorkomen 
3.	alle mutaties waarvan de ‘was’ en/of de ‘wordt’ in het interessegebied ligt

Er wordt geen speciaal oormerk toegevoegd als enkel de ‘was’- of de ‘wordt’ zich binnen het deelgebied bevindt. Afnemers herkennen deze situatie zelf en handelen deze op de juiste wijze af.

Toelichting:
Een “intredend object” waarvan enkel de ‘wordt’ in het interessegebied ligt, zal de afnemer als nieuw object gaan bijhouden.
Een “uittredend object” waarvan enkel de ‘was’ in het interessegebied ligt, zal de afnemer als object niet meer willen bijhouden.

Het gevolg is dat afnemers bij een “herintredend object” de mutatiehistorie missen van de periode waarin het object zich buiten hun interessegebied bevond. 

Om deze gedeeltelijke mutatiehistorie mee te kunnen leveren in deze situatie, vraagt zo veel dure centrale administratie en complexiteit dat dit wordt overgelaten aan de afnemer. Die kan bij de herstart van de bijhouding constateren dat het vroeger al bestaande object in het verleden is “uitgetreden” en zelf besluiten hoe hier mee om te gaan.
