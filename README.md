# How to build the manual
1. First you need to install sphinx (it is included in anaconda)
   You can install it through pip:
   ```bash
   $ pip install -U Sphinx
   ```

2. To build simply execute the following command in your terminal:
   ```bash
   $ sphinx-build <sourcedir> <outputdir>
   ```
3. To build the HTML files execute the following command in your terminal: 
   ```bash
   $ sphinx-build -b html <sourcedir> <outputdir>
   ```
4. To build PDF first execute the following command in your terminal:
   ```bash
   $ sphinx-build -b latex <sourcedir> <outputdir>
   ```
   Then change directory to `<outputdir>` and add `\DeclareUnicodeCharacter{2212}{-}` before `\begin{document}` in `GOMC.tex` file:
   Then run:
   ```bash
   $ make
   ```