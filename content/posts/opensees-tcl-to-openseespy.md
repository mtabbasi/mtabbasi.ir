---
title: "Convert an OpenSees(Tcl) script to OpenSeesPy"
description: "Convert an OpenSees(Tcl) script to OpenSeesPy"
date: 2023-07-20
draft: false
tags: ["opensees", "tcl", "python"]
customCss: ["/css/prism (3).css", "/css/prism-live.css", "/css/prism-line-numbers.css"]
customJs: [ "/js/prism (3).js", "/js/prism-live.js", "/js/prism-line-numbers.js"]
categories: ["app"]
cover: 
    image: "/img/2.png"
    alt: "opensees(tcl) to python converter"
    caption: ""
    hidden: false
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
<textarea oninput="convert()" class="prism-live line-numbers language-tcl fill"></textarea>

<h3>The openseespy result</h3>
<textarea id='python' class="prism-live line-numbers language-python fill"></textarea>



<script>
function isFloat(v) {
            if (isNaN(parseFloat(v))) {
                return false;
            }
            return true;
        }
function convert() {
    infile = document.querySelector("code.language-tcl").innerText;
    var ALIAS = "ops";
    const arrayRange = (start, stop, step) =>
        Array.from(
            { length: (stop - start) / step + 1 },
            (value, index) => start + index * step
        );
    var outfile = "import openseespy.opensees as ops\n\n";
    if (ALIAS.length > 0 && ALIAS.slice(-1) != ".") {
        ALIAS += ".";
    }
    for (let line of infile.split("\n")) {
        line = line.trim()
        let INFO = line.split(" ");
        INFO = INFO.filter(e => e);
        let N = INFO.length
        if (N < 1) {
            //Skip a blank line
            outfile += "\n";
            continue;
        }
        if (INFO[0][0] == "}") {
            //Ignore a close brace
            continue;
        }
        if (INFO[0][0] == "#") {
            //Echo a comment line
            outfile += line + "\n";
            continue;
        }
        if (INFO[0] == "print") {
            // Change print to printModel
            INFO[0] = "printModel"
        }

        if (N == 1) {
            //A command with no arguments, e.g., wipe
            outfile += `${ALIAS}${INFO[0]}()\n`;
            continue;
        }

        if (N >= 8 && ["nonlinearBeamColumn", "forceBeamColumn", "dispBeamColumn"].includes(INFO[1])) {
            // Needs to be a special case due to beam integration
            var eleTag = INFO[2];
            var secTag = INFO[6];
            /*
            The original element format
            element beamColumn tag ndI ndJ Np secTag transfTag
            0        1       2   3   4   5    6       7
            */
            if (isFloat(secTag)) {
                const Np = INFO[5];
                const transfTag = INFO[7];
                if (INFO[1] == "dispBeamColumn") {
                    outfile += `${ALIAS}beamIntegration('Legendre',${eleTag},${secTag},${Np})\n`;
                }
                else {
                    outfile += `${ALIAS}beamIntegration('Lobatto',${eleTag},${secTag},${Np})\n`;
                }

            }

            else {
                /*
                The newer element format
                element beamColumn tag ndI ndJ transfTag integrType ...
                0        1       2   3   4      5         6
                */
                const transfTag = INFO[5];
                outfile += `${ALIAS}beamIntegration('${INFO[6]}',${eleTag}`;
                for (let j of arrayRange(7, N - 1, 1)) {
                    outfile += `,${INFO[j]}`;
                }
                outfile += ")\n";
            }
            if (INFO[1] == "nonlinearBeamColumn") {
                info[1] = "forceBeamColumn";
            }
            outfile += `${ALIAS}element('${INFO[1]}',${eleTag},${INFO[3]},${INFO[4]},${transfTag},${eleTag})\n`;
            continue;
        }

        // Have to do the first argument before loop because of the commas
        if (isFloat(INFO[1])) {
            outfile += `${ALIAS}${INFO[0]}(${INFO[1]}`;
        } else {
            outfile += `${ALIAS}${INFO[0]}('${INFO[1]}'`;
        }

        // Now loop through the remaining arguments with preceding commas

        var writeClose = true;

        for (let i of arrayRange(2, N - 1, 1)) {
            if (INFO[i] == "{") {
                writeClose = true;
                break;
            }

            if (INFO[i] == "}") {
                writeClose = false;
                break;
            }

            if (isFloat(INFO[i])) {
                outfile += `,${INFO[i]}`;
            } else {
                outfile += `,'${INFO[i]}'`;
            }

        }

        if (writeClose) {
            outfile += ")\n";
        }

    }
    outfile += "\n\n";
    const tcl = document.querySelector("code.language-python");
    document.querySelector("code.language-python").innerHTML = highlight(outfile, 'python');
    
}

function highlight(code, language) {
 if (Prism.languages[language]) {
  return Prism.highlight(code, Prism.languages[language], language);
 } else {
  return Prism.util.encode(code);
 }
}
window.addEventListener("load", (event) => {
    let tcl = highlight(`
# units: kip, in

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
const el = document.querySelector("code.language-tcl");
el.innerHTML = tcl;
convert();
  
});
</script>

{{< /rawhtml >}}

## Source code [^2]

{{< gist mtabbasi 266b8b55de8fca9292c540f65c04c109 >}}

[^2]: [https://github.com/OpenSees/OpenSees/blob/master/SCRIPTS/toOpenSeesPy.py](https://github.com/OpenSees/OpenSees/blob/master/SCRIPTS/toOpenSeesPy.py)
