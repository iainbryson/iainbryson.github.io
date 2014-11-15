

# Emailing Client-Side Web Pages

### The Problem

Sending an email of a webpage is pretty easy.  That is, it's easy if the page is static or you're generating the webpage on the server (e.g. with PHP or ASP.NET).  If it's not -- if you have some client-side code which alters the page -- then it's not so easy.

This article discusses the paths I went along to mail pages from a fancy new HTML5-based client side web app.  The pages themselves were relatively static, but they were rendered using javascript based on data loaded via AJAX queries.

### Start with a Browser

Fortunately, we have the ideal tool to render web pages: the browser.  All major browsers are scriptable to some degree.  To send our page we'll use IE (10, specifically) and script it using powershell.

```powershell
$ie = new-object -com "InternetExplorer.Application"
$ie.visible = $true
$ie.navigate2($url)

# Wait for the page to start rendering
Start-Sleep -MilliSeconds 5500

$doc = $ie.Document

if (-not $doc) { Write-Error "Browser document is null"; exit(0); }

# Wait for the page to complete rendering
while ($ie.Busy -or $ie.Document.readyState -ne "complete") {
    Start-Sleep -MilliSeconds 100
    Write-Host $ie.Document.readyState
}
```

One unpleasant consequence of this is that evidently you must be running as administrator.  If you don't, you'll find the document member of the browser object to be null:

### Lack of Style

The first approach has one evident flaw: no styling.  The document we get from the IE DOM contains only the body and not the head elements.  Scripts we can do without; they're not going to be executed by the email client anyway.  But without stylesheets the page will be ugly.

So let's rebuild the stylesheets:

```powershell
$bodyHtml = "<!DOCTYPE html><html><head>";
foreach ($ss in $doc.styleSheets) {
	$href = $($ss | Select-Object -expandProperty href);
	Write-Output "Processing stylesheet $href";
	$bodyHtml += "<link rel='stylesheet' type='text/css' href='$($href)' />"
}
```

### Next Stumbling Block: Outlook

You'd think in 2012, with Microsoft's massive leaps forward in web standards conformance with IE that Outlook would use the same rendering engine and have no problems with basic CSS3.  You'd be [wrong](http://www.campaignmonitor.com/css/ "so, so wrong").  

So now we have to rewrite the webpage to use fewer bells and whistles, right?  Not necessarily.  We already have a browser up with our page and *it* certainly knows how to render the page.  Can we pass its knowledge along to Outlook?

With a bit of a hack we can.  The particular problem is that Outlook isn't rendering the CSS selectors properly.  Instead of relying on CSS to style the page, we can inject a script into IE that overwrites the style attributes with the computed style.  Essentially we're fixing the style into place.  This will bloat the page of course, but it'll render correctly.

So let's do that, and also remove any script:

```javascript
// Remove all the script on the page; we don't need to email it.
(function(){jQuery('script').remove();})();

$('body').append('<div id=""defaultElement""/>'); 

var hard_code_attributes = ['background-color', 'color', 'font-size', 'font-family'];

var defaultElement = $('#defaultElement').get()[0];

// add the computed style to the style attribute for every element.
// this prevents incorrect rendering with viewers which can't handle CSS3 (i.e. Outlook).
jQuery('*').each(function(e) {
    var newStyle = '';
    for (var i = 0; i < hard_code_attributes.length; i++) {
        var a = hard_code_attributes[i];

        if (this.currentStyle && this.currentStyle[a] && defaultElement.currentStyle && this.currentStyle[a] != defaultElement.currentStyle[a] ) {
            newStyle += a + ':' + this.currentStyle[a] + ';';
        }
    }
    this.setAttribute('style', newStyle);
});
```

### Final Stumbling Block: SVG

Next we have some pages with SVG.  This seems to be another area where Outlook's HTML mail rendering has difficulties.  

Fortunately, there's a solution.  Yet another messy solution, but one that works.  Google has a wonderful library called [canvg](http://code.google.com/p/canvg/ "canvg") which [renders SVG into a canvas](http://stackoverflow.com/questions/3975499/convert-svg-to-image-jpeg-png-etc-in-the-browser).  A canvas can then be exported to an image file as a [data url](http://stackoverflow.com/questions/923885/capture-html-canvas-as-gif-jpg-png-pdf):
 
### Hello

```javascript

$('body').append('<a href=""$($url)"">link</a>'); 

canvg();

// Render all the canvases as images.
$('canvas').each( function(d) {
    var img = this.toDataURL('image/png');
    var ii = $('<img src=`"'+img+'`"/>').attr('src',img);
    $(this).replaceWith(ii);
} ); 
}
```

One final snag: Outlook can't render large data urls. To get over size restrictions on data urls, we'll need to extract the base64 encoded data url into separate files and include it them as [attachments](http://social.technet.microsoft.com/Forums/en-US/winserverpowershell/thread/90ff6edd-75db-443a-bcaf-194f6f37e829 "helpful code").

```powershell
$attachments = @();

# Outlook won't render data url images as large as we need, so take the turn the data url into a separate file.
$imgNum = 1;
foreach ($img in $ie.Document.getElementsByTagName("img")) {
    $imgFileName = "img$($imgNum).png";
    $imgNum++;
    $t1 = $img.getAttribute("src");
    $txt = $t1.Replace("data:image/png;base64,", "");
    $img.setAttribute("src", "cid:$($imgFileName)");
    $bytes  = [System.Convert]::FromBase64String($txt);
    $decoded = [System.Text.Encoding]::Default.GetString($bytes); 
    [Byte[]]$bytes_imagefront=[System.Text.Encoding]::Default.GetBytes($decoded)
    set-content -encoding byte $imgFileName -value $bytes_imagefront
    $attachments += $imgFileName
}
```

Now all the pieces are in place, and we're ready to actually send the email.

```powershell
Send-MailMessage -To $to -From $from -BodyAsHtml -Body $bodyHtml -Subject $subject -SmtpServer smtphost -Attachments $attachments
```

