---
author: Benjamin R. Jack
date: "2016-06-11T00:00:00Z"
title: Embedding fonts into R-generated PDFs on OS X
---

I use R with ggplot2 to create publication-ready figures in PDF format. Using a combination of ggplot2 and cowplot, I rarely make any manual changes with Adobe Illustrator or any other vector-based editing tool. These script-generated PDF figures make it easy to modify my analyses without worrying about manually reformatting my figures. Unfortunately, when R generates a PDF, it does not embed the fonts. To understand why embedding fonts matters, compare the two plots below:

<img src="/assets/fonts-side-by-side.png" width="800" alt="A side-by-side comparison of a PDF with and without embedded fonts" />

The plot on the left appears exactly as it should, with all the type rendered with the Helvetica typeface. The plot on the right is a screenshot of a PDF rendered on a system that did not have Helvetica installed. Since R did not embed the fonts into the PDF, this system substitutes Helvetica with the closest font that it can find. The substitution is markedly different from the original Helvetica. Without embedded fonts, the same PDF renders very differently on different machines. Frustratingly, the systems that lack standard fonts like Helvetica tend to be archaic journal publishing systems. Overleaf also lacks the basic fonts to render PDFs correctly. To guarantee that PDFs render exactly the same across a variety of systems, we must embed the fonts into the PDF itself.

I wrote a short python script (adapted from [here](https://discussions.apple.com/message/28994467#message28994467)) that scans the working directory for PDFs, opens them in Preview, and re-saves them under the original filename. These re-saved PDF files have all fonts embedded. This script will only work on OS X, because it depends on OS X APIs.

{{< highlight python >}}
#!/usr/bin/python
import sys
from Quartz.PDFKit import PDFDocument
from Foundation import NSURL
for f in [ a.decode('utf-8') for a in sys.argv[1:] ]:
    PDFDocument.alloc().initWithURL_(NSURL.fileURLWithPath_(f)).writeToFile_(f)
{{< / highlight >}}

Calling the above script (which I've saved as `embed_fonts.py`) from within R has the effect of silently embedding fonts into R-generated PDFs.

{{< highlight R >}}
library(ggplot2)
library(cowplot)
myplot <- ggplot(iris, aes(x=Sepal.Length, y=Petal.Length, color=Species)) +
  geom_point() +
  labs(x="Sepal Length", y="Petal Length") +
  ggtitle("Plants")
save_plot("myplot.pdf", myplot)
# Embed fonts into every PDF in current working directory
system("./path/to/embed_fonts.py *.pdf")
{{< / highlight >}}

Now your PDFs will appear the same across any system, even systems without Helvetica installed.
