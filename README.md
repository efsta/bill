# Digitaler Kassenbon

## Zielsetzung
Als "Digitaler Kassenbon" ist hier die Technik beschrieben,
den durch den Händler an einer Computerkasse erstellten Verkaufsbeleg dem
Kunden sofort in elektronischer Form zur Verfügung zu stellen.

Mit "Verkaufsbeleg" ist primär die lesbare Darstellung der Transaktion
gemeint ('urschriftsgetreu' wie ein gedruckter Kassenbon, für Archivierung
gemäß gesetzlicher Vorschriften AO/BAO zumindest 'inhaltsgleich').

Auch wenn die Anzeige am Bildschirm die Einbettung aktiver Inhalte
(Links, Landing Page) erlaubt, darf das nicht den formalen Zweck als Dokument
(Buchhaltungs-Beleg) verschleiern oder behindern.

Und letztlich sollen die Detaildaten der Transaktion durch den Kunden ohne
Medienbruch gespeichert und weiterverarbeitet werden können.

## Die Rolle von efsta
Die "efsta IT Services GmbH" mit Sitz in Steyr, Österreich, bietet ein Subsystem
für Kassen an, das die ordnungsmäßige Sicherung der Verkaufstransaktionen
('Grundaufzeichnungen') vereinfacht.

Es besteht aus einem lokal an der Kasse laufenden Modul, das die länderspezifischen
Fiskal-Vorschriften (z.B. DE: KassenSichV, AT: RKS-V) technisch umsetzt, in 
Kombination mit Cloud-Services zur revisionssicheren Archivierung.

Neben diesen vom Händler zu erfüllenden Pflichten bietet efsta mit ihrer
Infrastruktur Services an, die mit dem Transport von Kassentransaktionen oder der
Bereitstellung aus dem Cloud-Archiv zu tun haben - dazu gehört der Digitale Kassenbon.

Nachdem efsta für viele Händler tätig ist, ist Datenschutz ein Grundgesetz:
in der Cloud werden alle Geschäftsdaten verschlüsselt gespeichert,
jede Transaktion hat einen eigenen Schlüssel. Daraus ergibt sich für die
Bereitstellung von Transaktionsdaten die Notwendigkeit, dass der Kunde
neben einem Zugriffsschlüssel über den Datensatzschlüssel verfügt.

Die unterschiedlichen Methoden, dem Kunden Belege bereitzustellen, sind im Abschnitt
"Usecases" beschrieben. Auch gibt es die Möglichkeit der Einbindung von
"Service-Providern", denen vom Händler oder von Kunden Schlüsselmaterial
überlassen wird, und die dann unverschlüsselte Daten verarbeiten können.
Hier ergeben sich aus Marketing-Sicht oder zur bequemeren, weiterführenden
Datennutzung eine Vielzahl an Geschäftsfeldern, die efsta selbst als
Infrastruktur-Anbieter nicht wahrnehmen darf.

## Wie kommt der Beleg zum Kunden
Um aus der Cloud dem Kunden Belege zu- oder bereitstellen zu können, müssen 
folgende Bedingungen erfüllt sein:

* 2C: der Kunde wird beim Verkaufsvorgang identifiziert und ist in der efsta Public
Key Infrastruktur (PKI) aktiviert, damit die Zugriffsdaten übermittelt werden
können
* 2P: es wird beim Verkaufsvorgang von einem Service-Provider (z.B. einem Zahlungs-
Dienstleister) eine Transaktionskennung vergeben, mit der in die Cloud
verschlüsselt und später auf den Beleg zugegriffen werden kann
* EXT: dem Kunden wird eine Transaktionskennung (z.B. die
Fiskal-Signatur in Österreich - QR-Code) übermittelt, z.B. in Form eines
Papier-Beleges oder als QR-Anzeige

Die Transaktionskennung ("Secret") muss
* hinreichend geheim sein
* hinreichend lang sein (Entropie)
* kann aus mehreren Feldern zusammengesetzt sein (z.B. Datum + Betrag + Abwicklungsnummer
Bezahldienstanbieter)

Es ist zu beachten, dass durch die alleinige QR-Anzeige für den Kunden beim
Verkaufsvorgang nicht die gesetzliche Belegerteilungs- und Mitnahmepflicht
erfüllt wird.

## Schlüssel in der efsta Cloud
### 2C Keyset
Die fixe Kunden-Zuordnung eines Belegs der Cloud erfolgt über
eine Belegkette. Diese wird in einer App bei Aktivierung festgelegt, dabei werden
folgende Felder in der Cloud hinterlegt (base64url):
* XID (256 bit): anonyme (zufällige) Kunden-ID
* PUBKEY (insgesamt 65 byte): Public Key (dzt. ECDH) für asymmetrische Verschlüsselung
* TIDM (256 bit): TID Modifier zur Berechnung der einzelnen Glieder der Belegkette

Der Pointer auf eine Einzel-Transaktion wird in der Cloud unter der Transaktions-ID
TID geführt, alle TIDs eines Kunden bilden die Glieder der Belegkette eines Keysets.
Der TID Startwert wird ermittelt aus

    TID[0] = hmac(TIDM, XID)
Danach wird die Kette mit

    TID[n+1] = hmac(TIDM, TID[n])
fortgeführt.

Im TID-Pointer wird auf die Transaktionstabelle verwiesen, sie enthält den
Datensatz-Index (Zugriff) und den asymmetrisch verschlüsselten Datensatz-Schlüssel
(zur Entschlüsselung am Endgerät).


## Usecases
Die nachfolgenden Usecases beschreiben Schlüsselverwaltung und Zugriffsmethoden
für den Digitalen Kassenbon. Die generelle Funktion muss aber durch den
Händler freigeschaltet sein (opt-in), und es muss eine Beleg-Formatierungsanweisung
("Layout") hinterlegt sein.

### UC_R2A Anonymous Customer
Der Kunde generiert mit einer App ein Keyset (bestehend aus zufällige XID, zufälligen TIDM und ein
zufälliges ECDH-Schlüsselpaar). Das Keyset wird - ausgenommen Private Key -
über das bill.efsta-API in der Cloud aktiviert.

Am Endgerät des Kunden muss das Keyset für den späteren Beleg-Zugriff gespeichert
werden. Für Backup ist zu sorgen, zumal ein verlorenes Keyset nicht wiederhergestellt
werden kann.

Gibt der Kunde beim Verkaufsvorgang seine XID bekannt, wird der
Beleg der Belegkette des Kunden angehängt und diesem sofort zugestellt.
Dazu ist nur bei Transaktions-Registrierung das Feld `ESR.XID` anzugeben.

Demo unter: bill.efsta.net/DemoR2A

### UC_R2C Customer Number
Entspricht weitgehend UC_R2A.

Nur wird die XID aus einer eindeutigen Kunden-Identifikation (Allocated ID) AID
im Format allocator/number ermittelt. Um global eindeutige XIDs zu erhalten, ist
der Domänenname als allocator zu verwenden.

Beispiel einer AID:

    AID = "mycompany.com/1234"
    XID = sha256(AID)        // hUwAu432uFQVdDJ-SnkwUwV9pkoCBnJS9u_gtbMaWzA

Die Registrierung erfolgt mit `ESR.XID = "mycompany.com/1234"` oder `ESR.XID = "hUw..."`.

In diesem Usecase wird ein Service-Provider ("Allocator") benötigt. Natürlich
kann der Händler selbst diese Rolle übernehmen und den Digitalen Beleg z.B. in
seiner Kunden-App bereitstellen.

### UC_EXT External Key Access
Wenn Kundenbelege aufgrund gesetzlicher Vorgaben immer eine den Anforderungen
entsprechende Transaktionskennung ("Secret") enthalten (z.B. Signatur), dann
wird unter dieser zusätzlich in der Tabelle EXT ein Transaktions-Pointer abgelegt:

    DecryptKey = sha256(Secret)
    EXT.RecordIndex = sha256(DecryptKey)

In Schritten (Zugriff auf EXT, Decrypt, Zugriff auf Transaktionstabelle DAT) kann
die Original-Transaktion geholt und entschlüsselt werden.
Der Datenschutz entspricht dem Prinzip "wer Secret kennt, hat den physischen Beleg".

Mit dieser Methode können direkt die Transaktionsdaten eines Beleges, den man in
Händen hält (und der in der Cloud gespeichert ist), ausgelesen werden. Besonders
einfach ist das bei maschinenlesbarer Darstellung (QR) des Secret.
