# How to build the manual
1. First you need to install sphinx (it is included in anaconda)
   You can install it through pip:
   ```bash
   $ pip install -U Sphinx
   ```

2. To build simply type:
   ```bash
   $ sphinx-build <sourcedir> <outputdir>
   ```

3. To build PDF first run the following command:
   ```bash
   $ sphinx-build -b latex <sourcedir> <outputdir>
   ```
   Then change directory to <outputdir> and add the following before `\begin{document}` in `GOMC.tex` file:
   `\DeclareUnicodeCharacter{2212}{-}`
   Then run:
   ```bash
   $ make
   ```