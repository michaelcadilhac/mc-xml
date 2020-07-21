
# Table of Contents

1.  [Usage](#org0ba1c4c)
    1.  [XML Format](#org4d5b134)
    2.  [TeX Template](#org85b5958)
    3.  [XML to TeX](#org1b5cd5f)
    4.  [TeX to D2L](#org5876590)
2.  [Requirements](#orgd88c670)

This package provides two tools:

-   `mc-xml2tex`: This takes as input an XML description of multiple-choice
    questions, and a TeX template, and lays out the questions within the
    template.  The script is in charge of randomizing the order of the questions
    and the answers, respecting constraints indicated in the XML file.  The TeX
    template should have a few lines of Python that receive a Question object, and
    prints it.

-   `mc-tex2xml`: This reads metadata from the TeX file about the questions
    (correct answers & points), then compiles the TeX file with a parameter.  The
    TeX file is in charge of isolating the question indicated in the parameter and
    only print that question.  The script then converts this output to PNG, and
    produces a CSV file that describes the quiz (with questions sourced from the
    PNG); that CSV in turn can be used as is in D2L.

The first tool can be used as is, but the second requires the first.


<a id="org0ba1c4c"></a>

# Usage


<a id="org4d5b134"></a>

## XML Format

The format used for questions is best presented with an example:

    <mc deltaq=2>
      <preamble>
        \usepackage{amsmath}
      </preamble>
      <question>
        This is the first question, but it could be moved around.  LaTeX can be freely used:
        $$x < 2 \Rightarrow p = \begin{pmatrix} 3 & 4 \\ 8 & 9\end{pmatrix}$$
        <choice correct>That is the correct answer, it could appear as A, B, or C.</choice>
        <choice>Incorrect answer appearing as A, B, or C.</choice>
        <choice>Incorrect answer appearing as A, B, or C.</choice>
        <choice fixed>Incorrect answer appearing as D (useful for ``none of the above'').</choice>
      </question>
    
      <question points=2 fixed>
        This is a question worth 2 points, which always appear in second position in
        the quiz (keyword \texttt{fixed}).
        <choice>Answer A, B, or C.</choice>
        <choice>Answer A, B, or C.</choice>
        <choice>Answer A, B, or C.</choice>
        <choice correct fixed>Answer D, which is correct.</choice>
      </question>
    
      <flush/>
    
      <question fixedanswers points=2>
        As we issued a \texttt{flush} just before, all the questions appearing
        before this one in the XML file will be output before it in the TeX output.
        Also, since this question has keyword \texttt{fixedanswers}, all the answers
        will appear in the order of the XML file.
        <choice>$L$ is undecidable.</choice>
        <choice>$L$ is Turing-recognizable, but not co-Turing-recognizable.</choice>
        <choice>$L$ is co-Turing-recognizable, but not Turing-recognizable.</choice>
        <choice correct>$L$ is regular.</choice>
      </question>
    
      <question fixedanswers onepar points=2 fixed>
        In this question, the answers are formated in a single paragraph, rather
        than each on their line.  It is the job of the TeX template to implement
        this flag; the script itsef has no notion of \text{onepar}.
        <choice correct>Yes.</choice>
        <choice>No.</choice>
        <choice>It is impossible to know.</choice>
        <choice>The question is not well-defined.</choice>
      </question>
    
      <question hideanswers fixedanswers>
        In this question, we explain answers A, B, C, D in the text, for some reason
        (my use case was some code where four lines where labeled A, B, C, and D).
        The TeX file will be in charge of not printing the choice of answers.
        (\texttt{fixedanswers} should thus be used---although, for historical
        reason, this is automatically the case.)
        <choice correct>A</choice>
        <choice>B</choice>
        <choice>C</choice>
        <choice>D</choice>
      </question>
    </mc>

The nodes and attributes that are understood by the scripts are:

-   `<mc deltaq=N>`: This indicates how far a question can end up, compared to
    its position in the XML file.  If `N` is 0, then the order in the XML file is
    the order in the TeX file (and hence the quiz).  If a `deltaq` is provided to
    the script `ml-xml2tex`, then it has precedence.
-   `<question points=2 fixed fixedanswers>`: `fixed` means that the position of
    the question in the XML file is the position in the TeX output (hence the
    quiz).  `fixedanswers` means that the *answers* to that question will appear
    in the order given in the XML file.  This is in particular useful when one
    answer is of the form &ldquo;Both A. and B.&rdquo;, and you need to be sure what A and B
    are.
-   `<choice correct fixed>`: Exactly one choice should have the `correct` flag.
    The `fixed` flag means that this answer should not be moved; this is useful
    for answers that you want to appear last, for instance &ldquo;None of the above&rdquo;.
-   `<flush/>`: This makes sure that all the questions before the tag are already
    printed.  This is useful when you have several topics in a quiz, and don&rsquo;t
    want to mix questions too much.  Also, if question `X` introduces a concept that
    is used in questions `Y` and `Z`, it is possible to ensure that this question
    appears before the others using:
    
        <mc>
          ...
          <flush/>
          <question fixed>X</question>
          <question>Y</question>
          <question>Z</question>
          ...
        </mc>
    
    or
    
        <mc>
          ...
          <question>X</question>
          <flush/>
          <question>Y</question>
          <question>Z</question>
          ...
        </mc>

The other attributes appearing in the example file (`onepar`, `hideanswers`)
must be interpreted by the template TeX file.


<a id="org85b5958"></a>

## TeX Template

Again, this is best presented with a minimal example.  This first template is a
minimal example for `mc-xml2tex`; we will see that if we plan to use
`mc-tex2quiz` afterward, the minimal template is slightly more complicated.

    \documentclass{article}
    
    %% For inparaenum.
    \usepackage{paralist}
    
    %!EXTRAPREAMBLE
    
    \begin{document}
    \begin{enumerate}
    
    %!BEGIN_QUESTIONS
    def isAttrTrue (elt, field):
        return elt.get (field, "false") != "false"
    print ("\\item " + question.text + "\n\n")
    if not isAttrTrue (question, "hideanswers"):
      if isAttrTrue (question, "onepar"):
        env = "inparaenum"
      else:
        env = "enumerate"
      print ("\\begin{" + env + "}[A.]")
      for ans in answers:
        print ("\\item " + ans.text)
        if isAttrTrue (ans, "correct"):
          print (" (correct)")
      print ("\\end{" + env + "}")
    points = question.get ("points", "1")
    print ("\\hfill (" + points + "pt" + \
                    ("s" if int (points) > 1 else "") + ")\n")
    %!END_QUESTIONS
    
    \end{enumerate}
    \end{document}

The script `mc-xml2tex` will:

1.  Print everything up to `%!EXTRAPREAMBLE`,
2.  Print the `preamble` node of the XML file (if any, see above),
3.  Print everything up to `%!BEGIN_QUESTIONS`,
4.  Use the Python snippet between `%!BEGIN_QUESTIONS` and `%!END_QUESTIONS`, to
    print questions,
5.  Print the rest of the TeX file.

The Python snippet reads the object `question` and the list `answers` and prints
them.  These are [`Element`](https://lxml.de/tutorial.html#the-element-class) objects, that is, the corresponding XML node.  The
main properties of interest are `text`, the actual text of the node, and the
`get()` method to retrieve attributes (`Element` behaves like a list).  The
children of the nodes are accessed with, e.g., `find()`, although this is not
needed in normal use.

In the example above, the Python snippet in the TeX template implements special
behavior for the flag `onepar` and `hideanswers`.

Further, if you plan on using `mc-tex2xml`, then the TeX file produces MUST read
the variable `\qnum`, and only print that question (ideally with the
`preview` package).  Here is a minimal example for this:

\documentclass{article}

%% For inparaenum.
\usepackage{paralist}

%% If there are no selected question, show everything.
\newif\ifnoqnum
\ifcsname qnum\endcsname
  \usepackage[active, tightpage]{preview}
  \setlength\PreviewBorder{0pt}%
  \noqnumfalse
\else
  \usepackage{preview}
  \noqnumtrue
\fi

%!EXTRAPREAMBLE

\begin{document}
\begin{enumerate}
\begin{preview}

%!BEGIN_QUESTIONS
def isAttrTrue (elt, field):
    return elt.get (field, "false") != "false"

## We count the number of questions, and if it matches \qnum, print it.
global nquestion
if not 'nquestion' in globals ():
  nquestion = 0
nquestion += 1

## Bypass the test if no \qnum were given, to print everything.
print ("\\ifnoqnum\\gdef\\qnum{" + str (nquestion) + "}\\fi")
## This if holds iff qnum = nquestion
print ("\\ifnum\qnum=" + str (nquestion) + "")

## Same as minimal.tex
print ("\\item " + question.text + "\n\n")
if not isAttrTrue (question, "hideanswers"):
  if isAttrTrue (question, "onepar"):
    env = "inparaenum"
  else:
    env = "enumerate"
  print ("\\begin{" + env + "}[A.]")
  for ans in answers:
    print ("\\item " + ans.text)
    if isAttrTrue (ans, "correct"):
      print (" (correct)")
  print ("\\end{" + env + "}")
points = question.get ("points", "1")
print ("\\hfill (" + points + "pt" + \
                ("s" if int (points) > 1 else "") + ")\n")

print ("\\fi")
%!END_QUESTIONS

\end{preview}
\end{enumerate}
\end{document}

The produced file 


<a id="org1b5cd5f"></a>

## XML to TeX

The usage is as follows:

    usage: mc-xml2tex [-h] [-d DELTAQ] FILE.xml TEMPLATE.tex
    
    Transform an XML Question file to TeX, randomizing questions and answers.
    
    positional arguments:
      FILE.xml              input file
      TEMPLATE.tex          template file
    
    optional arguments:
      -h, --help            show this help message and exit
      -d DELTAQ, --deltaquestions DELTAQ
                            how far from its original position can a question end
                            up, overrides the deltaq in the XML file, if any
                            (default: 0, in place)

The main nonobvious argument here is `QRAND`.  This indicates how far a question
can end up from its original position in the XML file.  The default, 0, means
that questions end up where they are in the file.  This is the same as having
all questions flagged with `fixed`.  This overrides the same setting in the XML
file.


<a id="org5876590"></a>

## TeX to D2L

The usage is as follows:

    usage: mc-tex2quiz [-h] [-b BASE_URL] [-o PICS_DIR] [-l LATEXMK] [-B BUILD_DIR] MC.tex QUIZ.csv
      Compiles each question in MC.tex to a PNG file, and creates a CSV quiz
      file for D2L.  The PNG are output in the PICS_DIR directory.
    
    Options:
      -b BASE_URL: The base URL where PNG files will be stored.
                   (default: https://michael.cadilhac.name/private/quiz/)
      -o PICS_DIR: The directory in which PNGs go.
                   (default: pics/)
      -l LATEXMK: latexmk command to use.
                   (default: latexmk -quiet)
      -B BUILD_DIR: The directory in which PDFs and aux files go before being converted to PNGs.
                   (default: _build/)
      -h: Prints this help message.
    
    Notes:
      This program relies on latexmk.  The file MC.tex MUST use the variable \qnum.

This will compile each question separately, using `latexmk`.  By default,
everything is compiled in the `pdf/` directory


<a id="orgd88c670"></a>

# Requirements

-   \(\LaTeX\) with a recent `latexmk`
-   Python3, with `numpy`, `scipy`.
-   `zsh`

