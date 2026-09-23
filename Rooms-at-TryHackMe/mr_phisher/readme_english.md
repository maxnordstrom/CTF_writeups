> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Mr. Phisher

https://tryhackme.com/room/mrphisher

![Screenshot](img/Pasted%20image%2020260226133755.png)

![Screenshot](img/Pasted%20image%2020260226133817.png)

Once we've spun up the machine there are two files

![Screenshot](img/Pasted%20image%2020260226133852.png)

The task instructions say "It keeps on asking me to "enable macros". What are those?" so I assume that's what happens when you open the document.

Correct, got this warning:

![Screenshot](img/Pasted%20image%2020260226134045.png)

The document contains an image

![Screenshot](img/Pasted%20image%2020260226134116.png)

The document is supposed to contain macros, and these can be found in LibreOffice via **Tools > Macros > Edit Macros**

![Screenshot](img/Pasted%20image%2020260226134946.png)

Now it's just a matter of figuring out how the code works! But first, let's see if Libre can run the macro directly. Checking in **Options** and I see I can change the security level. Time to allow all macros!

![Screenshot](img/Pasted%20image%2020260226144600.png)

Then **Run Macro**. Here goes nothing...

![Screenshot](img/Pasted%20image%2020260226145120.png)

Nothing seems to happen at the moment, as far as I can tell.

After a bit of searching it seems the code is written in VBA (Visual Basics for Applications). Not a language I'm fluent in, so I'll try to find a cheat sheet.

I've gathered that LibreOffice uses the Basic language, but that with `Option VBASupport 1` certain VBA can be used. `Dim` declares a variable, and the array with all the numbers is probably the decimal form of various characters (no number is above 127).

Let CyberChef decode the array just to take a look.

It gets a bit messy though since 127 is in there (DEL) along with a bunch of numbers below 32 which aren't printable ASCII characters.

![Screenshot](img/Pasted%20image%2020260226151519.png)

The code also contains `Xor`, so the numbers in the array probably shouldn't be decoded straight off. After a bit more searching I think I understand the code.

```python
Rem Attribute VBA_ModuleType=VBAModule # Rem is a comment
Option VBASupport 1 # Provides VBA support
Sub Format()
Dim a() # Declares the variable a as an array
Dim b As String # Declares the variable b as a string

# Here the array is assigned to the variable a
a = Array(102, 109, 99, 100, 127, 100, 53, 62, 105, 57, 61, 106, 62, 62, 55, 110, 113, 114, 118, 39, 36, 118, 47, 35, 32, 125, 34, 46, 46, 124, 43, 124, 25, 71, 26, 71, 21, 88)

# A for loop where i starts at 0 and iterates over the array a
For i = 0 To UBound(a)

# In the loop, something is appended to the variable b.
# This something is an ASCII character (which Chr produces 
# based on a number)
# This number is taken from the array a at index i, and this number
# is then run through Xor with i.
b = b & Chr(a(i) Xor i)

# Next makes the loop go again from the start
Next

# When the loop is done, the process ends
End Sub
```

I figured I should be able to add a line that prints out `b`. Seems unnecessary to me to rewrite the code in some other language... A quick google gives me:

> **Output to the Immediate Window (Debug.Print)**: Use `Debug.Print` to display the value during debugging.

However, I got an error message when I added it...

![Screenshot](img/Pasted%20image%2020260226153521.png)

Trying the other suggestion, `MsgBox b`, which should print the variable `b` in a popup. That worked great!

<details>
  <summary><b>Click here to see the flag</b></summary>

  ![Screenshot](img/Pasted%20image%2020260226153623.png)
</details><br>

And that was the correct flag!
