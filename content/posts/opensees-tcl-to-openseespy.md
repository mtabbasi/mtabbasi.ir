---
title: "Convert an OpenSees(Tcl) script to OpenSeesPy"
description: "Convert an OpenSees(Tcl) script to OpenSeesPy"
date: 2023-07-20
draft: false
tags: ["opensees", "tcl", "python"]
customCss: ["https://pyscript.net/latest/pyscript.css", "/css/prism (3).css", "/css/prism-live.css", "/css/prism-line-numbers.css"]
customJs: ["https://pyscript.net/latest/pyscript.js", "/js/prism (3).js", "/js/prism-live.js", "/js/prism-line-numbers.js"]
categories: ["app"]
---

:warning: **Before using this app please checkout [notes](#notes)**

## Notes

> Assumes the OpenSees(.tcl) files defines commands line by line
> without any loops, conditionals, variables, expressions, etc.
> This is the format generated when you export a model from
> OpenSees Navigator and perhaps from other front-ends to OpenSees.[^1]

> If your OpenSees(.tcl) files do use loops, conditionals, variables,
> expressions, etc., you might be better off to port your OpenSees
> model from Tcl to Python manually, or you can look in to Tkinter[^1]

> You may have some luck making your own "middleware" to convert your
> OpenSees(.tcl) script to a model defined line by line by inserting
> output statements in your loops and other constructs.  Even though
> this won't get you to 100% conversion and you'll still have some
> conversions to make here and there, it'll get you pretty far.[^1]

[^1]: The above quote is excerpted from [Michael H. Scott's](https://github.com/mhscott) [script](https://github.com/OpenSees/OpenSees/blob/master/SCRIPTS/toOpenSeesPy.py) in opensees repository.

## Tcl to python converter

### Enter your tcl code with considering [notes](#notes) in below box

{{< rawhtml >}}
<textarea  py-input="convert()" class="prism-live line-numbers language-tcl fill"></textarea>


<h3>The openseespy result</h3>
<textarea id='python' class="prism-live line-numbers language-python fill"></textarea>

<py-script>
import js
from js import highlight


def isfloat(value):
    try:
        float(value)
        return True
    except ValueError:
        return False

def convert():
    alias = "ops"
    outfile = "import openseespy.opensees as ops\n\n"
    infile = js.document.querySelector("code.language-tcl").innerText
    # Add a dot if needed
    if len(alias) > 0 and alias[-1] != ".":
        alias = alias + "."

    for line in infile.splitlines():
        info = line.split()
        N = len(info)

        # Skip a blank line
        if N < 1:
            outfile += "\n"
            continue
            # Ignore a close brace
        if info[0][0] == "}":
            continue
        # Echo a comment line
        if info[0][0] == "#":
            outfile += line
            continue

        # Change print to printModel
        if info[0] == "print":
            info[0] = "printModel"

        # A command with no arguments, e.g., wipe
        if N == 1:
            outfile += f"{alias}{info[0]}()\n"
            continue

        # Needs to be a special case due to beam integration
        if N >= 8 and info[1] in [
            "nonlinearBeamColumn",
            "forceBeamColumn",
            "dispBeamColumn",
        ]:
            eleTag = info[2]
            secTag = info[6]
            # The original element format
            # element beamColumn tag ndI ndJ Np secTag transfTag
            #    0        1       2   3   4   5    6       7
            if isfloat(secTag):
                Np = info[5]
                transfTag = info[7]
                if info[1] == "dispBeamColumn":
                    outfile += (
                        f"{alias}beamIntegration('Legendre',{eleTag},{secTag},{Np})\n"
                    )

                else:
                    outfile += (
                        f"{alias}beamIntegration('Lobatto',{eleTag},{secTag},{Np})\n"
                    )

            # The newer element format
            # element beamColumn tag ndI ndJ transfTag integrType ...
            #    0        1       2   3   4      5         6
            else:
                transfTag = info[5]
                outfile += f"{alias}beamIntegration('{info[6]}',{eleTag}"
                for j in range(7, N):
                    outfile += f",{info[j]}"
                outfile += ")\n"
            if info[1] == "nonlinearBeamColumn":
                info[1] = "forceBeamColumn"
            outfile += f"{alias}element('{info[1]}',{eleTag},{info[3]},{info[4]},{transfTag},{eleTag})\n"
            continue

        # Have to do the first argument before loop because of the commas
        if isfloat(info[1]):
            outfile += f"{alias}{info[0]}({info[1]}"
        else:
            outfile += f"{alias}{info[0]}('{info[1]}'"
        # Now loop through the remaining arguments with preceding commas
        writeClose = True
        for i in range(2, N):
            if info[i] == "{":
                writeClose = True
                break
            if info[i] == "}":
                writeClose = False
                break
            if isfloat(info[i]):
                outfile += f",{info[i]}"
            else:
                outfile += f",'{info[i]}'"
        if writeClose:
            outfile += ")\n"

    outfile += "\n\n"
    js.document.querySelector("code.language-python").innerHTML = highlight(outfile, 'python')

</py-script>

<script>
function highlight(code, language) {
 if (Prism.languages[language]) {
  return Prism.highlight(code, Prism.languages[language], language);
 } else {
  return Prism.util.encode(code);
 }
}
window.addEventListener("load", (event) => {
    let tcl = highlight(`# units: kip, in

# Remove existing model
wipe

# Create ModelBuilder (with two-dimensions and 2 DOF/node)
model BasicBuilder -ndm 2 -ndf 2

# Create nodes
# ------------
# Create nodes & add to Domain - command: node nodeId xCrd yCrd
node 1   0.0  0.0
node 2 144.0  0.0
node 3 168.0  0.0
node 4  72.0 96.0

# Set the boundary conditions - command: fix nodeID xResrnt? yRestrnt?
fix 1 1 1
fix 2 1 1
fix 3 1 1

# Define materials for truss elements
# -----------------------------------
# Create Elastic material prototype - command: uniaxialMaterial Elastic matID E
uniaxialMaterial Elastic 1 3000

#
# Define elements
#

# Create truss elements - command: element truss trussID node1 node2 A matID
element Truss 1 1 4 10.0 1
element Truss 2 2 4 5.0 1
element Truss 3 3 4 5.0 1

# Define loads
# ------------
#

# create a Linear TimeSeries with a tag of 1
timeSeries Linear 1

# Create a Plain load pattern associated with the TimeSeries,
# command: pattern Plain $patternTag $timeSeriesTag { load commands }

pattern Plain 1 1 {
    # Create the nodal load - command: load nodeID xForce yForce
    load 4 100 -50
}`, 'tcl');
document.querySelector("code.language-tcl").innerHTML = tcl;
  
});
</script>

{{< /rawhtml >}}


## Source code [^2]
{{< gist mtabbasi 266b8b55de8fca9292c540f65c04c109 >}}

[^2]: [https://github.com/OpenSees/OpenSees/blob/master/SCRIPTS/toOpenSeesPy.py](https://github.com/OpenSees/OpenSees/blob/master/SCRIPTS/toOpenSeesPy.py)