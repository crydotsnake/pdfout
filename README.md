# PDF out – dompdf und pdfjs für REDAXO

PDF out stellt den "HTML to PDF"-Converter [dompdf](http://dompdf.github.io) und [pdf.js](https://github.com/mozilla/pdf.js) in REDAXO zur Verfügung.
Mit dompdf können Ausgaben in REDAXO als PDF generiert werden und mittels pdf.js angezeigt werden.  
PDF out ist keine "out of the box"-Lösung. Es stellt nur die nötigen Vendoren bereit.  
___


## PDF-Generierung mittels dompdf

### Anforderungen

- DOM-Erweiterung
- MBString-Erweiterung
- php-font-lib
- php-svg-lib

> Man sollte beachten, dass einige erforderliche Abhängigkeiten weitere Abhängigkeiten haben können (insbesondere php-svg-lib erfordert sabberworm/php-css-parser).

### Empfehlungen

OPcache (OPcache, XCache, APC, etc.): verbessert die Leistung
GD (für Bildverarbeitung)
IMagick- oder GMagick-Erweiterung: verbessert die Bildverarbeitungsleistung
Besuchen Sie das Wiki für weitere Informationen: https://github.com/dompdf/dompdf/wiki/Requirements

### Mögliche Anwendung

Nach der Installation und Aktivierung kann ein PDF wie folgt erzeugt werden:
- Den nachfolgenden Code am Anfang des gewünschten Templates oder als separates Template einsetzen
- Der Aufruf erfolgt dann über die Variable pdf=1 die über die URL übergeben wird. Der aktuelle Artikel kann so dann als PDF ausgegeben werden. 

Sofern dann an eine aufgerufenen URL **?pdf=1** angehängt wird, wird der Inhalt von REX_ARTICLE[] oder REX_TEMPLATE [] als PDF ausgegeben.

### Demo

[diese Seite als PDF](index.php?pdftest=1)

**Hinweis**
- Der Aufruf klappt nur über das REDAXO Backend.
- Wenn man hinter die Backend url `?pdftest=1` dranhängt, kommt die README vom Addon.

## Beispiel-Code

```php
<?php
  // ?pdf=1
  $print_pdf = rex_request('pdf', 'int');
  if ($print_pdf) {
	  // Artikel laden oder alternativ ein Template
	  $pdfcontent = 'REX_ARTICLE[]';
	  // Outputfilter auf Inhalt anwenden, sofern erforderlich 
	  // Wenn nicht, wird die Generierung beschleunigt
	  $pdfcontent = rex_extension::registerPoint(new rex_extension_point('OUTPUT_FILTER', $pdfcontent));

	      // Dompdf konfigurieren
	      $dompdf = new Dompdf\Dompdf($options);
	      $dompdf->set_option('defaultFont', 'Helvetica');
	      $dompdf->set_option('dpi', '100');
	      $dompdf->set_option('font_cache', rex_path::addonCache('pdfout', 'fonts'));
	      $dompdf->setPaper('A4', 'portrait');

	      // Inhalte laden und rendern
	      $dompdf->loadHtml($pdfcontent);
	      $dompdf->render();

	      // Datei als PDF-Download ausliefern
	      $art_pdf_name =  rex_string::normalize(rex_article::getCurrent()->getValue('name'));
	      rex_response::sendResource($dompdf->output(), 'Content-Type: application/pdf', null, null, null, 'attachment', $art_pdf_name); 
	      
	}
?>
```
### Erweitertes Beispiel mit inline-css und URL-Ersetzung

Damit Bilder ausgegeben werden können, müssen die Bild-URLs umgeschrieben werden. MediaManager-URLs können nicht sofort genutzt werden. Die Bilder müssen direkt aus dem `media/`-Ordner ausgelesen werden oder mit voller URL angegeben werden. (siehe hierzu: https://github.com/FriendsOfREDAXO/pdfout/issues/13). Unbedingt die Kommentare beachten.

**Exemplarische Logik**

```php
if (true === $html) {
	echo rex_url::media($file); // URL für den Browser
} else if (true === $pdf) {
	echo rex_path::media($file); // URL für dompdf
}
```

> Tipp: Wenn möglich, dann Bilddateien als JPG innerhalb eines `<img>`-Tags platzieren. Dies bietet die beste Performance.

Externes CSS kann im `<head>` eingebunden werden.

```php
<?php
$print_pdf = rex_request('pdf', 'int');
// ?pdf=1 
if ($print_pdf) {
	$pdfcontent = 'REX_ARTICLE[]';
	// Outputfilter auf Inhalt anwenden, sofern erforderlich
	// Wenn nicht, wird die Generierung beschleunigt
	$pdfcontent = rex_extension::registerPoint(new rex_extension_point('OUTPUT_FILTER', $pdfcontent));
	// Hier Beispiele für Image-Rewrite
	// Bei der Verwendung von MediaManager-Bildern anpassen    
	$pdfcontent = str_replace("/index.php?rex_media_type=standard&amp;rex_media_file=", "media/", $pdfcontent);
	$pdfcontent = str_replace("index.php?rex_media_type=redactorImage&amp;rex_media_file=", "media/", $pdfcontent);
	$pdfcontent = str_replace("index.php?rex_media_type=redactorImage&rex_media_file=", "media/", $pdfcontent);
	// übliche Links in das Medienverzeichnis    
	$pdfcontent = str_replace("/media/", "media/", $pdfcontent);
	$pdfcontent = str_replace(".media/", "media/", $pdfcontent);

	// Kopfdefinition
	$pre = '
	<head>

	</head>
	<body>
	<img id="titel" src="media/logo.jpg" style="width:100%"; height:auto;" />
	<style>
	/* hier CSS anlegen, Beispiel: */

	body { 
	    padding-left: 80px; 
	    padding-top:10px; 
	    line-height: 1.56em; 
	    }
	a {
	    display: none;
	    } 
	 h1 {font-size: 2.5em; font-color: #990000;}

	</style>
	';
	      // Dompdf konfigurieren
	      $dompdf = new Dompdf\Dompdf($options);
	      $dompdf->set_option('defaultFont', 'Helvetica');
	      $dompdf->set_option('dpi', '100');
	      $dompdf->set_option('font_cache', rex_path::addonCache('pdfout', 'fonts'));

	      $dompdf->setPaper('A4', 'portrait');
	      // Inhalte laden und rendern
	      $dompdf->loadHtml($pre.$pdfcontent.'</body>');
	      $dompdf->render();
	      // Datei als PDF-Download ausliefern.
	      $art_pdf_name =  rex_string::normalize(rex_article::getCurrent()->getValue('name'));
	      rex_response::sendResource($dompdf->output(), 'Content-Type: application/pdf', null, null, null, 'attachment', $art_pdf_name); 
	      
	      
  }
?>
```

##  Media-Manager-Bilder alternative Einbindung

Wesentlich ist hierbei eine Einstellung im Template und das Einfügen der Server-URL, was man auch via Output Filter lösen könnte. 

**Modul**:
```php
<?php 

$imgs = explode(",","REX_MEDIALIST[1]");
foreach ($imgs as $img) {
    echo "<img src='".rex::getServer()."index.php?rex_media_type=pdf-optimized&rex_media_file=".$img."'><br>";
}
?>
```

**Template**:
Habe das Beispiel-Template verwendet. Hier ist der Teil ab Zeile 39 relevant. Neu ist die Option `$options->set('isRemoteEnabled', TRUE);` - damit wird offenbar das Abrufen von Remote-URL aktiviert. 

```php
// Dateiname 
$art_pdf_name =  rex_string::normalize(rex_article::getCurrent()->getValue('name'));
header('Content-Type: application/pdf');
$options = new Dompdf\Options();
$options->set('defaultFont', 'Helvetica');
$options->set('isRemoteEnabled', TRUE);
$options->set('font_cache', rex_path::addonCache('pdfout', 'fonts'));
$dompdf = new Dompdf\Dompdf($options);
$dompdf->loadHtml($pre.$pdfcontent.'</body>');
$dompdf->setPaper('A4', 'portrait');
$dompdf->render();
$dompdf->stream($art_pdf_name ,array('Attachment'=>false));
die();
```

IMHO könnte man die Anweisungen am Anfang des Templates entsprechend umschreiben, dass die lokale URL via `rex::getServer()` vorangestellt werden: `$pdfcontent = str_replace("/index.php?rex_media_type=standard&amp;rex_media_file=", "media/", $pdfcontent);`

Weitere Infos:  https://github.com/dompdf/dompdf/issues/1118
___
## Tipps
- Es sollte auf die numerische Angabe bei font-weight verzichtet werden. 
- Es empfiehlt sich im verwendeten Template die CSS-Definitionen nicht als externe Dateien sondern inline zu hinterlegen. Dies beschleunigt die Generierung, da keine externen Ressourcen eingelesen werden müssen.
- Auf Bootsstrap CSS oder andere CSS-Frameworks bei der Ausgabe möglichst verzichten, da zuviele Styles abgearbeitet werden müssen. 
- URLs zu Ressourcen sollten ohne / beginnen und vom Webroot aus definiert sein z.B. media/zyz.jpg oder assets/css/pdf_styles.css. Ein Search & Replace per PHP kann hierbei helfen. 
- Fixierte Divs können zur Anzeige von Fuß und Kopfzeile verwendet werden. Ideal ist es diese direkt nach dem Bodytag zu integrieren. Dann können auch mittels CSS count z.B. Seitenzahlen ausgegegeben werden.
- Google Fonts zur lokalen Nutzung herunterladen: https://google-webfonts-helper.herokuapp.com/fonts

## Font-Awsome 4.x einbinden: 
Font-Awsome fonts werden nicht korrekt dargestellt. 
Folgender Workaround hilft: 
Einbindung z.B. CDN im Template

Zusätzlichen Stil in Style-Tag inline einfügen: 

```html	
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/font-awesome/4.6.3/css/font-awesome.min.css">
	<style>
	.fa {
	    display: inline;
	    font-style: normal;
	    font-variant: normal;
	    font-weight: normal;
	    font-size: 14px
	    line-height: 1;
	    color: #2F2ABD;
	    font-family: FontAwesome;
	    font-size: inherit;
	    text-rendering: auto;
	    -webkit-font-smoothing: antialiased;
	    -moz-osx-font-smoothing: grayscale;
	  }
	</style>  
```

## Ausgabe eines PDF mit pdf.js

### Link-Beispiel

```html
<a href="<?= rex_url::assets('addons/pdf_viewer/vendor/web/viewer.html?file=/media/deinePDFdatei.pdf') ?>">Link</a> 
```
### Tipp
Möchte man dompdf-urls  der andere URLs mit Parametern kombinieren, muss die übergebene Url unbedingt per urlencode kodiert werden. `urlencode($foo)`

Also z.B.: 

```html
<a href="<?= rex_url::assets('addons/pdfout/vendor/web/viewer.html?file='.urlencode("index.php?pdftest=1"))?>

```

## Lead
- [Thomas Skerbis](https://github.com/skerbis)


___
### Credits

- [dompdf](http://dompdf.github.io)
- [FriendsOfREDAXO](https://github.com/FriendsOfREDAXO)
- [First release: Oliver Kreischer](https://github.com/olien)
- [Simon Krull](https://github.com/crydotsnake)
- [Alexander Walther](https://github.com/alexplusde)
