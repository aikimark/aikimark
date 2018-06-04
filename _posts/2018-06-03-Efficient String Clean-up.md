---
layout: post
title: Efficient String Clean-up
description: An analysis of various means of cleaning strings of multiple space characters
tags: tech regex
---
<h2>Introduction</h2>
<p>I frequently answer questions where strings need to be 'cleaned' of multiple space characters.&nbsp;The most common fix is to remove leading or trailing spaces.&nbsp;For that problem, there are very handy intrinsic VB functions (LTrim, RTrim,
 Trim).&nbsp;However, these functions do not affect any repeated space characters between the first and last non-space characters -- the internal spaces.</p>
<p>&nbsp;</p>
<p>In this article, I will explore different solutions to this problem and evaluate their performance. You should be able to apply this in any of the Office products (Access, Excel, PowerPoint, Word), VBScript, or VB6. However, the collection object is not
 available in the VBScript environment, so you won't be able to use or test any of the methods that use the collection object in your VBS batch jobs.</p>
<p>&nbsp;</p>
<p><strong>Note:</strong> You can use these methods to remove any repeating characters inside a string, not just repeated space characters.</p>
<p>&nbsp;</p>
<h2>Problem Context</h2>
<p>You often have little say in the data you must process.&nbsp;Count yourself lucky if you get well-formed, normalized, and structured data.&nbsp;However, if you receive any of the following, this article may help you.</p>
</p>
<ul>
 <li>unstructured data (text files) that you will need to parse or format</li>
 <li>text or memo (varchar) database fields with raw data</li>
 <li>HTML or XML that may look good in a browser, but isn't easy to parse or format</li>
 <li>XML or JSON (any tagged data format) that you need to compact before sending or saving</li>
</ul>
<h2>Performance Methodologies</h2>
<p>The performance of the measured methods is affected by the length of the strings and the amount of (internal) space characters to be removed.&nbsp;For simplicity, I will evaluate the code and methods with different permutations on the Gettysburg Address
 -- one paragraph, the entire address, ten copies of the address concantenated in one string, and one hundred copies of the address concantenated in one string.&nbsp;A description of the permutation routine is at the end of this article.</p>

<p>In addition to different permutations of the Gettysburg Address, I am measuring the code against both the string permutations and a version of each string permutation with additional inserted space characters between the words. A description of the space-insertion
 routine is at the end of this article.</p>
<p>I'm showing different methods in order of simplicity of the code, with the intrinsic VB functions (<strong>Replace, Split, Join</strong>) being covered first before the ActiveX objects (<strong>Regexp, WorksheetFunction.Trim</strong>).&nbsp;In the
 Performance Results and comparisons section, I order the methods from fastest to slowest.&nbsp;I will include some explanation of why these behave differently.</p>
<p>Although I used different timing methods in my tests, I only include the most precise method in the attached sample code. When timed statements in the code run very quickly, sub millisecond, most of the easiest-to-code timing methods fail to recognize
 that any time has passed.</p>
<p>It is always difficult to get reliable timing data from a Windows OS.&nbsp;There are so many things happening with other applications, utilities, and services,&nbsp;that you need to give yourself permission to ignore outliers. In these tests, I am only
 concerned with normal execution values.&nbsp;Upon the advice of a stats friend, I used the median function. This eliminates spikes in the results. The median value is the value that is middle value, or average of the two middle values in a sorted list
 of values.</p>
<p>Before each test, I set the priority of the Windows process to High and closed all non-essential applications. As part of the launching process, the application window is minimized. I walked away from my laptop for the 5-7 minutes required for each test
 run.</p>
<h2>Preparation of the data</h2>
<p>Here are the steps I took to measure the algorithms.</p>
<ol>
 <li>Make three separate 'performance test' executions</li>
 <li>Each performance test creates four different permutation lengths of the address.</li>
 <li>For each address permutation, the set of algorithms are invoked against the plain permutation and a 'space-stuffed' version of the address.</li>
 <li>Invoke each algorithm 21 times</li>
 <li>Calculate the median of each algorithm</li>
 <li>Normalize each algorithm's median against the median of an Instr() for that iteration's permutation (plain or stuffed).&nbsp;The Instr() function is looking for a string that does not exist in that iteration's string.</li>
 <li>The three normalized 'performance test' execution sets are averaged</li>
 <li>The averaged (normalized) results are sorted by each test's overall average.</li>
</ol>
<h2>Gettysburg Address Data Profile</h2>
<p>The four permutations of the address provide a wide range of data profiles for the code to process. &nbsp;Without any additional internal spaces inserted, the profiles look like this:</p>
```  
Permutation     	Words	Non-space chars	Total chars
First paragraph  	   30  	   146        	   175
Entire Gburg Addr	  271 	  1186        	  1464
Gburg Addr x10  	 2710	 11860        	 14640
Gburg Addr x100 	27100	118600        	146400
```
<h2>Simple Replace -- [just two]</h2>
<p>Don't fall into the trap that a single invocation of the Replace() function will reliably remove all instances of repeated characters.&nbsp;You need to iterate the Replace() function until there are no more repeated characters.</p>
``` vb  
Function JustTwo(ByVal parmString As String) As String
    '====================================
    'Replace all double space strings with a single space.
    'Iterate until there are no more double space character
    'strings
    '====================================
    Dim strTemp As String
    strTemp = parmString
    Do Until InStr(strTemp, "  ") = 0
        strTemp = Replace(strTemp, "  ", " ")
    Loop
    JustTwo = strTemp
End Function
```  
<h2>Ambitious Replace - [three &&nbsp;two]</h2>
<p>Now that we know we have a simple, reliable, and fast method it is time to ask if it can go faster.&nbsp;Obviously, the answer is "yes".&nbsp;Otherwise, I wouldn't have been able to write a very interesting and educational article.&nbsp;In
 the following approach, I add a second loop that will first change all strings with three consecutive spaces and then change all the strings with two consecutive spaces.</p>
``` vb
Function ThreeTwo(ByVal parmString As String) As String
    '================================================
    'Replace all three consecutive spaces with one space, 
    'then replace all two consecutive spaces with one space
    '================================================
    Dim strTemp As String
    strTemp = parmString

    'Replace three space strings with a single space until
    'no more instances of three space strings exist
    Do Until InStr(strTemp, "   ") = 0
        strTemp = Replace(strTemp, "   ", " ")
    Loop

    'Replace two space strings with a single space until no 
    'more instances of two space strings exist
    Do Until InStr(strTemp, "  ") = 0
        strTemp = Replace(strTemp, "  ", " ")
    Loop
    ThreeTwo = strTemp
End Function
```
<h2>Complex Replace - [multiple replaces]</h2>
<p>Let's kick the <strong>Ambitious Replace</strong> approach up a notch...BAM! If you are familiar with the Shell sort, this should be a somewhat familiar algorithm. We attempt to replace some longer space character sequences before shorter ones. This
 algorithm extends the Ambitious Replace algorithm beyond just three and two length space sequences. If you know something about your data, you might get great results by optimizing the string sizes you replace.</p>
``` vb
Function MultiLengths(ByVal parmString As String, _
                     ByVal parmLengths As Variant) As String
    '==================================
    'Iterate the parmLengths array and invoke the Replace() function with a space string
    'of each length.
    '==================================
    Dim vItem As Variant
    Dim strTemp As String
    Dim strFind As String
    
    strTemp = parmString
    For Each vItem In parmLengths
        strFind = Space(vItem)    'create vItem length space string
        Do Until InStr(strTemp, strFind) = 0
            strTemp = Replace(strTemp, strFind, " ")
        Loop
    Next
    MultiLengths = strTemp
End Function
```
<h2>Intrinsic function performance</h2>
<p>Before we look at the Split() methods, it might help to know what part of the overall elapsed time is taken up by just the Instr(), Split(), and Join() functions.</p>
<p>Max InStr() perf:</p>
```
Max (sec) 	Min      	Median    	Avg     	Q3
0.000003283	0.000001676	0.000002095	0.000002165	0.000002165	Plain 175
0.000005029	0.000004540	0.000004889	0.000004872	0.000004959	Plain 1464
0.000034990	0.000031010	0.000033873	0.000033730	0.000034292	Plain 14640
0.000322806	0.000295429	0.000320222	0.000318942	0.000321130	Plain 146400

0.000003492	0.000002584	0.000002933	0.000002910	0.000002933	Stuffed 175
0.000012083	0.000011105	0.000011594	0.000011617	0.000011733	Stuffed 4700
0.000108883	0.000098127	0.000104622	0.000104429	0.000105181	Stuffed 47759
0.001042451	0.001010883	0.001034070	0.001032131	0.001037632	Stuffed 477271
```
<p>Performance relative to the Max Instr():</p>
```
            	Plain 175	Plain 1464	Plain 14640	Plain 146400
Join() perf:	1.9     	 3.7     	 4.4     	 5.9
Split2 perf:	2.5     	 5.2     	 6.7     	 7.2
Split perf: 	5.7     	17.6     	22.9     	24.6

            	Stuffed 514	Stuffed 4700	Stuffed 47759	Stuffed 477271
Join() perf:	 1.5     	 1.5         	 1.4         	   1.9
Split2 perf:	18.2     	39.3        	47.5        	 398.0
Split perf: 	35.5     	79.4        	96.6        	1793.5
```
<p>The fact that the InStr() function universally performed best in all tests prompted me to use its performance as a normalization value for the measured code results. Also, note the Split2 performance (delimiter = two space characters) is between 2 and
 4 times faster than the Split performance (delimiter = single space character).</p>
<h2>Simple Split &&nbsp;Join - [split &&nbsp;join]</h2>
<p>The Split() function is an easy to use and useful parsing function.&nbsp;Unfortunately, it does not have the capability to ignore repeated delimiters. So, we must iterate the Split() results, looking for non-zero-length items.</p>
``` vb
Function SplitJoin(ByVal parmString As String) As String
    '================================================
    'Split() string with single space character delimiter,
    'add non-empty strings to the strWords array.
    'Then Join() the strWord array items with
    'a single space character
    '================================================
    Dim strWords() As String
    Dim strParsed() As String
    Dim vItem As Variant
    Dim lngLoop As Long
    Dim lngWord As Long
    Dim strtemp As String
    
    strParsed = Split(parmString, " ")
    ReDim strWords(0 To UBound(strParsed))
    lngLoop = 0
    lngWord = 0
    
    'Add non-empty strings to strWord array
    For lngLoop = 0 To UBound(strParsed)
        strtemp = strParsed(lngLoop)
        If Len(strtemp) <> 0 Then
            strWords(lngWord) = strtemp
            lngWord = lngWord + 1
        End If
    Next

    'reduce size of the strWords array to equal the number
    'of non-empty strings we found.
    ReDim Preserve strWords(0 To lngWord - 1)
    SplitJoin = Join(strWords, " ")

End Function
```
<p><strong>Note:</strong> My performance measurement of the parsed array iteration prompted me to replace the For Each&hellip;Next loop with the traditional For...Next loop. &nbsp;It was measurably faster.</p>

<h2>Split2 &&nbsp;Join - [split2 &&nbsp;join]</h2>
<p>While the SimpleSplit method does work, it isn't the most efficient algorithm.&nbsp;Looking at the performance test results, I realized that simplicity doesn't always result in the fastest code. An apt analogy is the simplicity of very simple
 sorting algorithms that perform quite badly with non-trivial amounts of data. In this algorithm, I split on a string of two spaces. There is a trade-off with this method.&nbsp;In order to process correctly, I have to Trim() any leading or trailing spaces
 from the Split() function results.</p>

``` vb
Function Split2Join(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitJoin, but using double space character
    'string as delimiter for the Split() function
    '
    'Split() string with single space character delimiter,
    'move non-empty strings to the front of the strParsed array,
    'Redim the strParsed array down to the number of words we have,
    'then Join() the strParsed array items with
    'a single space character
    '================================================
    Dim strParsed() As String
    Dim lngLoop As Long
    Dim lngWord As Long
    Dim strtemp As String
    
    strParsed = Split(parmString, "  ")
    lngWord = 0
    
    'Move non-empty strings to the front of the strParsed array
    For lngLoop = 0 To UBound(strParsed)
        strtemp = strParsed(lngLoop)
        If Len(strtemp) <> 0 Then
            strParsed(lngWord) = strtemp
            lngWord = lngWord + 1
        End If
    Next

    'reduce size of the strParsed array to equal the number
    'of non-empty strings we found.
    ReDim Preserve strParsed(0 To lngWord - 1)
    Split2Join = Join(strParsed, " ")

End Function
```
<h2>Split and concatenate - [split &&nbsp;concat]</h2>
<p>If you have relatively short strings, this might be an alternative to the Split and Join approaches above.&nbsp;However, string concatenation can quickly become a performance beast as you can see in the performance comparison section of this article.</p>
``` vb
Function SplitConcat(ByVal parmString As String) As String
    '================================================
    'Split() string with single space character delimiter, 
    'concatenate non-empty strings to the returned value
    '================================================
    Dim strParsed() As String
    Dim strTemp As String
    Dim lngLoop As Long
    strParsed = Split(parmString, " ")
    lngLoop = 0
    strTemp = vbNullString

    For lngLoop = 0 To UBound(strParsed)
        strTemp = strParsed(lngLoop)
        If Len(strTemp) <> 0 Then
            SplitConcat = SplitConcat & strtemp & " "     'concatenate with space
        End If
    Next
    SplitConcat = RTrim(SplitConcat)

End Function
```
<h2>Split2 and concatenate - [split2 &&nbsp;concat]</h2>
 <p>Here, I do the same two-space split with the required Trim() of extra spaces.&nbsp;While this does perform better than the single-space Split(), the concatenation operations kill performance with larger strings.</p>
``` vb
Function Split2Concat(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitConcat, but using double space character
    'string as delimiter for the Split() function
    '
    'Split() string with a double space character delimiter,
    'concatenate non-empty strings to the returned value
    '================================================
    Dim strParsed() As String
    Dim strtemp As String
    Dim lngLoop As Long
    strParsed = Split(parmString, "  ")
    strtemp = vbNullString
    
    For lngLoop = 0 To UBound(strParsed)
        strtemp = Trim(strParsed(lngLoop))
        If Len(strtemp) <> 0 Then
            Split2Concat = Split2Concat & strtemp & " "     'concatenate with space
        End If
    Next
    Split2Concat = RTrim(Split2Concat)      'remove trailing space

End Function
```
<h2>Split2 and buffer - [Split2 &&nbsp;buffer]</h2>
 <p>Here, I do the same two-space split with the required Trim() of extra spaces.&nbsp;Buffering is a technique, using the Mid() function, that provides a fast alternative to concatenation.</p>
 <p><strong>NOTE:</strong> This can not be done in the VBScript environment.</p>
``` vb
Function Split2Buffer(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitBuffer, except using a double space
    'delimiter for the Split() function.
    '
    'Split() string with a double space character delimiter,
    'assign non-empty strings to next output buffer position,
    'returned the trimmed output buffer string
    '================================================
    Dim strParsed() As String
    Dim strTemp As String
    Dim lngLoop As Long
    Dim lngWordPosn As Long
    Dim strBuffer As String
    
    strParsed = Split(parmString, "  ")
    strTemp = vbNullString
    
    lngWordPosn = 1
    strBuffer = Space(Len(parmString))
    For lngLoop = 0 To UBound(strParsed)
        strTemp = Trim(strParsed(lngLoop))
        If Len(strTemp) <> 0 Then
            Mid$(strBuffer, lngWordPosn, Len(strTemp)) = strTemp
            lngWordPosn = lngWordPosn + Len(strTemp) + 1
        End If
    Next
    Split2Buffer = RTrim(strBuffer)

End Function
```
<h2>Split2 and buffer to the function variable - [Split2BufferFcn]</h2>
 <p>Here, I do the same two-space split with the required Trim() of extra spaces.&nbsp;Buffering is a technique, using the Mid() function, that provides a fast alternative to concatenation. In this test, I use the function return string value instead of
  a local string variable for the buffering.</p>
 <p><strong>NOTE:</strong> This can not be done in the VBScript environment.</p>
``` vb
Function Split2BufferFcn(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitBuffer, except using a double space
    'delimiter for the Split() function.
    '
    'Split() string with a double space character delimiter,
    'assign non-empty strings to next output buffer position,
    'returned the trimmed output buffer string
    '================================================
    Dim strParsed() As String
    Dim strTemp As String
    Dim lngLoop As Long
    Dim lngWordPosn As Long
    
    strParsed = Split(parmString, "  ")
    strTemp = vbNullString
    
    lngWordPosn = 1
    Split2BufferFcn = Space(Len(parmString))
    For lngLoop = 0 To UBound(strParsed)
        strTemp = Trim(strParsed(lngLoop))
        If Len(strTemp) <> 0 Then
            Mid$(Split2BufferFcn, lngWordPosn) = strTemp
            lngWordPosn = lngWordPosn + Len(strTemp) + 1
        End If
    Next
    Split2BufferFcn = RTrim(Split2BufferFcn)

End Function
```
<h2>Split into collection and Join - [split &&nbsp;col]</h2>
 <p>The VB collection object is very efficient for storing strings, especially when you don't know how many strings you need to store. We can also add the non-empty strings (words) into a collection object. In order to use the Join() function, we still
  have to populate an array. For clarity, I used a different array, strWords, rather than strParsed.</p>
``` vb
Function SplitCol(ByVal parmString As String) As String
    '================================================
    'Split() string with single space character delimiter,
    'adding the non-empty strings to a collection object.
    'Copy the collection items to an array and
    'Join() them as the returned value
    '================================================
    Dim strParsed() As String
    Dim strtemp As String
    Dim lngLoop As Long
    Dim strWords() As String
    Dim colWords As New Collection
    Dim vItem As Variant
    
    strParsed = Split(parmString, " ")
    For lngLoop = 0 To UBound(strParsed)
        strtemp = strParsed(lngLoop)
        If Len(strtemp) <> 0 Then
            colWords.Add strtemp
        End If
    Next
    ReDim strWords(1 To colWords.Count)
    lngLoop = 1
    For Each vItem In colWords
        strWords(lngLoop) = vItem
        lngLoop = lngLoop + 1
    Next
    SplitCol = Join(strWords, " ")

End Function
```
<h2>Split2 into collection and Join - [split2 &&nbsp;col]</h2>
 <p>The only difference between this test and the SplitCol() test above is the use of a double space delimiter for the Split() function.</p>
``` vb
Function Split2Col(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitCol, except using a double space
    'delimiter for the Split() function.
    '
    'Split() string with double space character delimiter,
    'adding the non-empty strings to a collection object.
    'Copy the collection items to an array and
    'Join() them as the returned value
    '================================================
    Dim strParsed() As String
    Dim strtemp As String
    Dim lngLoop As Long
    Dim strWords() As String
    Dim colWords As New Collection
    Dim vItem As Variant
    
    strParsed = Split(parmString, "  ")
    For lngLoop = 0 To UBound(strParsed)
        strtemp = Trim(strParsed(lngLoop))
        If Len(strtemp) <> 0 Then
            colWords.Add strtemp
        End If
    Next
    ReDim strWords(1 To colWords.Count)
    lngLoop = 1
    For Each vItem In colWords
        strWords(lngLoop) = vItem
        lngLoop = lngLoop + 1
    Next
    Split2Col = Join(strWords, " ")

End Function
```
<h2>Split into dictionary and Join - [split &&nbsp;dic]</h2>
 <p>With the collection object approach (above), we still have to transfer the words into the strWords array in order to use the Join() function. However, if we use a dictionary, an ActiveX object, we can apply the Join() function directly to the dictionary
  object's Items array.</p>
``` vb
Function SplitDic(ByVal parmString As String) As String
    '================================================
    'Split() string with a single space character delimiter,
    'adding non-empty strings to the dictionary,
    'then Join() the dictionary object's items array.
    '================================================
    Dim strParsed() As String
    Dim lngLoop As Long
    Dim lngKey As Long
    Dim strtemp As String
    
    Static dicWords As Object
    If dicWords Is Nothing Then
        Set dicWords = CreateObject("scripting.dictionary")
    Else
        dicWords.RemoveAll
    End If
    strParsed = Split(parmString, " ")
    lngKey = 1
    
    For lngLoop = 0 To UBound(strParsed)
        strtemp = strParsed(lngLoop)
        If Len(strtemp) <> 0 Then
            dicWords.Add CStr(lngKey), strtemp
            lngKey = lngKey + 1
        End If
    Next
    SplitDic = Join(dicWords.items, " ")

End Function
```
<h2>Split2 into dictionary and Join - [split2 &&nbsp;dic]</h2>
 <p>In this approach, we split with a double-space instead of a single space, populating a dictionary object with the non-empty strings.</p>
``` vb
Function Split2Dic(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitDic, but using double space character
    'string as delimiter for the Split() function
    '
    'Split() string with a double space character delimiter,
    'adding non-empty strings to the dictionary,
    'then Join() the dictionary object's items array
    '================================================
    Dim strParsed() As String
    Dim lngLoop As Long
    Dim lngKey As Long
    Dim strtemp As String
    
    Static dicWords As Object
    If dicWords Is Nothing Then
        Set dicWords = CreateObject("scripting.dictionary")
    Else
        dicWords.RemoveAll
    End If
    strParsed = Split(parmString, "  ")
    lngKey = 1
    
    For lngLoop = 0 To UBound(strParsed)
        strtemp = Trim(strParsed(lngLoop))
        If Len(strtemp) <> 0 Then
            dicWords.Add CStr(lngKey), strtemp
            lngKey = lngKey + 1
        End If
    Next
    Split2Dic = Join(dicWords.items, " ")

End Function
```
<h2>The Regular Expression Object</h2>
 <p>The regular expression ActiveX object is a very powerful tool for parsing text and pattern matching.&nbsp;It also has the ability to perform find/replace operations. While the intrinsic VB functions normally outperform regexp methods, there are some
  instances where the regexp object really shines. If you are unfamiliar with the regexp object, there is a link to an excellent introductory article in the references section.</p>
 <p>Since ActiveX objects take some time to instantiate, I measured two different ways of using the regexp object, minimizing the instantiation and pattern compilation overhead. The first way is to pass the regexp object into a function. The second way is
  to use a static variable in the function.</p>
 <p>Although I tested three regexp patterns, only the first two should be used for removing duplicate space characters. The third pattern might also remove other non-visible characters, such as tabs, carriage returns, and line feeds. I included this last
  pattern in order to measure the overhead of looking for any non-visible character against looking for just the space character.</p>
 <p><strong>The routine for the passed regexp object:</strong></p>
``` vb
Function RegexpReplace(ByVal parmString As String, parmRegexp As Object) As String
    '================================================
    'Use parameter regexp object to remove duplicate spaces.
    'The parameter regexp object will already have its pattern property set
    'by the calling code.
    '================================================
    RegexpReplace = parmRegexp.Replace(parmString, " ")
End Function
```
<h3>Regexp replace -- [regexp ' &nbsp;'+] and [RegexpReplace1]</h3>
 <p><strong>Pattern:</strong> " +"</p>
 <p><strong>Matches:</strong> a space character followed by one or more space characters.</p>
```  vb
Function RegexpReplace1(ByVal parmString As String) As String
    '================================================
    'Use local static regexp object to remove duplicate spaces
    '================================================
    Static oRE As Object
    If oRE Is Nothing Then
        Set oRE = CreateObject("vbscript.regexp")
        oRE.Global = True
        oRE.Pattern = "  +"
    End If
    RegexpReplace1 = oRE.Replace(parmString, " ")
End Function
```
 <h3>Regexp replace -- [regexp ' '{2,}] and [RegexpReplace2]</h3>
 <p><strong>Pattern:</strong> " {2,}"</p>
 <p><strong>Matches:</strong> two or more space characters.</p>
``` vb
Function RegexpReplace2(ByVal parmString As String) As String
    '================================================
    'Use local static regexp object to remove duplicate spaces
    '================================================
    Static oRE As Object
    If oRE Is Nothing Then
        Set oRE = CreateObject("vbscript.regexp")
        oRE.Global = True
        oRE.Pattern = " {2,}"
    End If
    RegexpReplace2 = oRE.Replace(parmString, " ")
End Function
```
 <h3>Regexp replace -- [regexp ' '\s+] and [RegexpReplace3]</h3>
 <p><strong>Pattern:</strong> " \s+"</p>
 <p><strong>Matches:</strong> Look for a space character, Chr(32), followed by one or more 'space class' characters. &nbsp;The 'space class' characters are any of the following [space, tab, carriage return, line feed, form feed].</p>
``` vb
Function RegexpReplace3(ByVal parmString As String) As String
    '================================================
    'Use local static regexp object to remove duplicate spaces
    '
    'WARNING: This pattern will remove characters other than
    '       space characters due to the use of the \s in the pattern
    '================================================
    Static oRE As Object
    If oRE Is Nothing Then
        Set oRE = CreateObject("vbscript.regexp")
        oRE.Global = True
        oRE.Pattern = " \s+"
    End If
    RegexpReplace3 = oRE.Replace(parmString, " ")
End Function
```
 <h3>Regexp parse-- [regexp parse]</h3>
 <p>In this approach, I'm using the Regular expression object to parse the different words in the string and then reconstructing the address with a single space character between each word with the Join() function.</p>
 <p><strong>Pattern:</strong> "[]^ ]+"</p>
 <p><strong>Matches:</strong> sequences of one or more non-space characters. This should preserve the words, punctuation, carriage returns, and line feed characters.</p>
``` vb
Function RegexParse(ByVal parmString As String) As String
    '================================================
    'Use local static regexp object to parse the words.
    'Copy the parsed words to an array and Join them with
    'a single space delimiter
    '================================================
    Dim oMatches As Object
    Dim oM As Object
    Dim strWords() As String
    Dim lngLoop As Long
    Static oRE As Object
    
    If oRE Is Nothing Then
        Set oRE = CreateObject("vbscript.regexp")
        oRE.Global = True
        oRE.Pattern = "[^ ]+"
    End If
        
    Set oMatches = oRE.Execute(parmString)
    ReDim strWords(0 To oMatches.Count - 1)
    lngLoop = 0
    For Each oM In oMatches
        strWords(lngLoop) = oM.Value
        lngLoop = lngLoop + 1
    Next
    RegexParse = Join(strWords, " ")
End Function
```
 <h3>Regexp parse&nbsp;-- [RegexParseBuffer]</h3>
 <p>In this approach, I'm using the Regular expression object to parse the different words in the string and then reconstructing the address with a single space character between each word with a buffering technique using the Mid() function. This buffering
  technique is a faster alternative to concatenation.</p>
 <p><strong>NOTE:</strong> This can not be done in the VBScript environment.</p>
 <p><strong>Pattern:</strong> "[]^ ]+"</p>
 <p><strong>Matches:</strong> sequences of one or more non-space characters. This should preserve the words, punctuation, carriage returns, and line feed characters.</p>
``` vb
Function RegexParseBuffer(ByVal parmString As String) As String
    '================================================
    'Use local static regexp object to parse the words.
    'Copy the parsed words to the buffer
    '================================================
    Dim oMatches As Object
    Dim oM As Object
    Static oRE As Object
    Dim strBuffer As String
    Dim lngWordPosn As Long
    Dim strTemp As String
    
    If oRE Is Nothing Then
        Set oRE = CreateObject("vbscript.regexp")
        oRE.Global = True
        oRE.Pattern = "[^ ]+"
    End If
        
    Set oMatches = oRE.Execute(parmString)
    strBuffer = Space(Len(parmString))
    
    lngWordPosn = 1
    For Each oM In oMatches
        strTemp = oM.Value
        Mid$(strBuffer, lngWordPosn, Len(strTemp)) = strTemp
        lngWordPosn = lngWordPosn + Len(strTemp) + 1
    Next
    RegexParseBuffer = RTrim(strBuffer)
End Function
```
 <h2>The WorksheetFunction Object</h2>
 <p>The Excel application object has a collection of WorksheetFunction methods. These are the intrinsic functions that you can use in your cell formulas (with a few exceptions). You can also use these functions in your VBA and VBScript environments. When
  removing internal duplicate spaces, you can use the WorksheetFunction.Trim() method. Thanks to Patrick Matthews for alerting me to the existence of this function. When I had used it in the past, I thought I had been using the VBA Trim() function. Using
  this function comes with limitations and warnings. As a result, I discourage its use outside of the Excel environment when the length of the string might exceed 32K. The testing code will prevent the execution of this function, both native and COM,
  when the string is too large.</p>
 <ul>
  <li>The maximum string length allowed is 32K, which is the maximum cell content length.</li>
  <li>In non-Excel environments, you have to instantiate an Excel.Application object before you can use any of the WorksheetFunction.Trim() function.</li>
  <li>There is measurable overhead going through the COM interface of the Excel.Application object.</li>
 </ul>
 <h3>
  <br>The Class code -- [clsXL trim]</h3>
 <p>To make it easier to use, I placed all the relevant code in a class. This ensures that the Excel.Application object is freed from memory when your application ends.</p>
``` vb
Option Explicit

Dim oXL As Object
Dim fnTrim As Object

Private Sub Class_Initialize()
    Set oXL = CreateObject("Excel.Application")
    Set fnTrim = oXL.WorksheetFunction
End Sub

Private Sub Class_Terminate()
    Set fnTrim = Nothing
    oXL.Quit
    Set oXL = Nothing
End Sub

Public Function CleanInternalSpaces(ByVal parmString As String) As String
    CleanInternalSpaces = fnTrim.Trim(parmString)
End Function
```
 <h3>The WorksheetFunction.Trim() code - [WksFunc Trim]</h3>
 <p>You do not need to place the WorksheetFunction.Trim() invocation inside a function. I placed it inside a function for a better side-by-side performance comparison with the code in the class module.</p>
``` vb
Function WksFunctionTrim(ByVal parmString As String) As String
    '================================================
    'If running in the Excel VBA environment, invoke the
    'Trim Worksheetfunction
    '================================================
    WksFunctionTrim = WorksheetFunction.Trim(parmString)
End Function
```
 <h2>Performance Results and Comparisons</h2>
 <p>Let's take a look at the performance results before we get into the analysis and recommendations.</p>
 <h2>Plain Data - Just permutations/slices of the address</h2>
 <p>In these <em>Plain</em> tests, there aren't any internal double space character strings to remove. These test results let us see the costs of invoking these routines unnecessarily.</p>
```
Method      	Plain 175	Plain 1464	Plain 14640	Plain 146400	Plain Avg
RegexpReplace3	  3.7     	  4.1     	  4.2     	   4.9         	   4.2
RegexpReplace1	  4.2     	  4.2     	  4.2     	   4.8         	   4.3
regexp ' \s+'	  5.0     	  4.5     	  4.4     	   5.0         	   4.7
RegexpReplace2	  4.0     	  4.9     	  5.4     	   6.1         	   5.1
just two    	  2.6     	  5.2     	  7.0     	   7.5         	   5.5
regexp ' {2,}'	  5.8     	  6.0     	  5.7     	   6.1         	   5.9
regexp '  +'	  8.7     	  6.5     	  4.8     	   4.7         	   6.2
Split2 & concat	  4.7     	  6.8     	  8.2     	   9.9         	   7.4
Split2 & Join	  5.5     	  7.2     	  8.6     	   9.2         	   7.6
Split2BufferFcn	  4.9     	  8.6     	 11.0     	  12.7         	   9.3
Split2 & buffer	  5.0     	  8.8     	 11.2     	  13.0         	   9.5
three & two  	  3.6     	  9.4     	 13.1     	  13.9         	  10.0
Split2 & col	  8.8     	  9.5     	 10.0     	  12.5         	  10.2
WksFunc Trim	 12.8     	 12.2     	  8.0     	            	  11.0
Split2 & dic	 11.4    	 10.2     	 10.6     	  12.0         	  11.0
Multiple repls	 10.2     	 24.2     	 32.3     	  33.4         	  25.0
Split & buffer	 17.8     	 59.5     	 83.5     	  89.3         	  62.5
Split & Join	 22.1     	 73.9     	101.7     	 111.4         	  77.3
Split & col 	 47.0     	142.7     	197.7     	 207.6         	 148.7
RegexParseBuf	 48.8     	167.5     	238.7     	 244.5         	 174.9
clsXL trim  	335.0     	182.7     	 45.8     	            	 187.8
regexp parse	 60.6     	192.6     	271.3     	 277.9         	 200.6
Split & dic 	 82.9     	273.8     	396.3     	 525.4         	 319.6
Split & concat	 25.6     	100.4     	400.4     	4594.7         	1280.3
```
 <p>There is such a wide range of (relative performance) values that we need to use a log scale when displaying all the methods. &nbsp;The worst performers go off the top side of the chart.</p>
 <p>
  <a href="https://filedb.experts-exchange.com/incoming/2015/01_w05/894473/PlainPerfChartAll.png"><img alt="PlainPerfChartAll.png" src="https://filedb.experts-exchange.com/incoming/2015/01_w05/800_894473/PlainPerfChartAll.png" class="fr-dib" style="max-width: 800px;"></a>If we only look at the best performers, we can use a linear scale.
  <a href="https://filedb.experts-exchange.com/incoming/2015/01_w05/894474/PlainPerfChartBest.png"><img alt="PlainPerfChartBest.png" src="https://filedb.experts-exchange.com/incoming/2015/01_w05/800_894474/PlainPerfChartBest.png" class="fr-dib" style="max-width: 800px;"></a>&nbsp;</p>
 <h2>Stuffed Data - Stuff those slices with extra spaces</h2>
```
Method      	Stuffed 514	Stuffed 4700	Stuffed 47759	Stuffed 477271	Stuffed Avg
RegexpReplace2	  6.0     	  7.1         	  7.3         	   9.9         	   7.6
RegexpReplace1	  6.4     	  7.4         	  7.4         	  10.0         	   7.8
regexp ' {2,}'	  6.9     	  7.9         	  7.5         	  10.2         	   8.1
RegexpReplace3	  6.4     	  8.3         	  8.6         	  11.3         	   8.7
regexp ' \s+'	  7.2     	  8.6         	  8.7         	  11.2         	   8.9
regexp '  +'	 10.3     	  9.0         	  7.9         	  10.1         	   9.3
WksFunc Trim	 10.9     	  8.6         	            	               	   9.7
RegexParseBuff	 39.3     	 75.8         	 83.5         	  87.5         	  71.5
regexp parse	 48.1     	 86.4         	 93.3         	  95.7         	  80.9
Multiple repls	 58.8     	111.2         	116.1         	 125.0         	 102.8
three & two 	 39.0     	 82.0         	 94.3         	 211.1         	 106.6
clsXL trim  	254.5     	 92.6         	            	              	 173.6
Split2BufferFcn	 64.1     	141.3         	165.0         	 525.6         	 224.0
Split2 & buffer	 64.9     	143.4         	167.3         	 522.1         	 224.4
Split2 & Join	 69.9    	150.1         	172.1         	 522.6         	 228.7
just two    	 64.1     	138.2         	160.1         	 563.7         	 231.5
Split2 & col	 82.8     	177.6         	204.0         	 553.2         	 254.4
Split2 & dic	106.8     	228.8         	263.8         	 651.2         	 312.6
Split2 & concat	 71.5     	164.7         	305.1         	1877.6         	 604.7
Split & buffer	119.9     	271.8         	321.9         	1993.2         	 676.7
Split & Join	124.8     	273.5         	325.1         	2020.8         	 686.1
Split & col 	141.4     	305.4         	359.1         	2025.7         	 707.9
Split & dic 	167.2     	365.2         	422.6         	2153.2      	 777.0
Split & concat	127.1     	289.7         	461.5         	3176.0         	1013.6
```
 <p>When we look at the performance of the methods doing actual work, we still have to use a log scale.</p>
 <p>&nbsp;
  <a href="https://filedb.experts-exchange.com/incoming/2015/01_w05/894475/StuffedPerfChartAll.png"><img alt="StuffedPerfChartAll.png" src="https://filedb.experts-exchange.com/incoming/2015/01_w05/800_894475/StuffedPerfChartAll.png" class="fr-dib" style="max-width: 800px;"></a>For the best performers (regexp replace and WorksheetFunction.Trim methods),
  a narrow linear scale shows they have very similar performance profiles.</p>
 <p>
  <a href="https://filedb.experts-exchange.com/incoming/2015/01_w05/894476/StuffedPerfChartBest.png"><img alt="StuffedPerfChartBest.png" src="https://filedb.experts-exchange.com/incoming/2015/01_w05/800_894476/StuffedPerfChartBest.png" class="fr-dib" style="max-width: 800px;"></a>&nbsp;</p>
 <h2>Performance Analysis</h2>
 <p>The plain permutations show us that there is a non-trivial cost to some of these algorithms.</p>
 <p>&nbsp;</p>
 <p>The stuffed permutations show us that memory management and string handling can cause algorithms to behave very badly.</p>
 <p>&nbsp;</p>
 <h2>Performance Recommendations</h2>
 <p>Here's a list of recommendations</p>
 <ul>
  <li>Check to see if there is anything to be done, no matter what algorithms and functions you use. &nbsp;Instr() is fast, so it is worth the overhead.</li>
  <li>Although the <strong>vbscript.regexp</strong> object isn't normally as fast as the Split() function, the simplicity of the Split() function, and limited pattern options, causes it to be slower than regexp when the pattern isn't implemented.</li>
  <li>The Replace() approach is usually faster than splitting the string, with the Regexp replace function much faster for repeated character removal.</li>
  <li>Trim() and Join() are also very fast functions.</li>
  <li>Avoid string concatenation when you are faced with the possibility of long strings, using Join() or the buffering technique.</li>
  <li>The use of collections and dictionaries won't make up for the inefficiencies of the Split() function when removing duplicate character strings.</li>
  <li>Clear out or reset variables when timing</li>
  <li>When creating log files, think about how the data will be used and make your parsing tasks easier.</li>
  <li>Validate your code. &nbsp;Take a unit testing approach and verify that what you are testing actually produces correct/expected results.</li>
  <li>Local variables perform better than repeatedly altering the function value.</li>
  <li>Local object variables perform better than parameterized/passed objects.</li>
  <li>When iterating arrays, the standard For...Next loop is faster than the For Each...Next loop.</li>
 </ul>
 <p>&nbsp;</p>
 <h2>Inserting spaces</h2>
 <p>When the space characters are inserted into each permutation, there can be between one and 26 spaces between each word in the address, with a trend towards an average of 13 consecutive spaces.</p>
``` vb
Function StuffWithSpaces(ByVal parmString As String, parmSeed) As String
    '================================================
    'Add Random number of internal space characters to the address permutation
	'Since I am specifying a max space length of 26, the average space sequence
	'will be 13 characters long.
    '================================================
    Dim lngRnd As Long
    Dim strWords() As String
    Dim lngLoop As Long
    Const cMaxSpaces As Long = 26
    
    Rnd -1					'reset the random sequence
    Randomize parmSeed		'initialize the random sequence
    strWords = Split(parmString, " ")
    For lngLoop = 0 To UBound(strWords) - 1
        lngRnd = Int(Rnd() * cMaxSpaces) + 1
        strWords(lngLoop) = strWords(lngLoop) & Space(lngRnd)
    Next

    StuffWithSpaces = Join(strWords, vbNullString)	'don't add any more spaces with the
													'Join() operation
End Function
```  
<h2>Code and Files</h2>
 <p>The text of the Gettysburg Address: &nbsp;<a class="file-inline" href="https://filedb.experts-exchange.com/incoming/2015/01_w03/892491/GettysburgAddress.txt">GettysburgAddress.txt</a></p>
 <p>The log file from a test run: &nbsp;<a class="file-inline" href="https://filedb.experts-exchange.com/incoming/2015/01_w05/894477/DeSpaceLog.txt">DeSpaceLog.txt</a></p>
 <p>A parsed and massaged version of the log file with statistics: &nbsp;<a class="file-inline" href="https://filedb.experts-exchange.com/incoming/2015/01_w05/894478/DeSpaceLog.xls">DeSpaceLog.xls</a></p>
``` vb
Option Explicit

Private Declare Function getTickCount Lib "kernel32" Alias "GetTickCount" () As Long

Private Declare Function CPUFrequency Lib "kernel32" _
    Alias "QueryPerformanceFrequency" (cyFrequency As Currency) As Long

Private Declare Function CPUTickCount Lib "kernel32" _
    Alias "QueryPerformanceCounter" (cyTickCount As Currency) As Long
    
Enum eSizeRequest
    eFirstParagraph = 1
    eSameAsDocument = 2
    eTenFold = 3
    eHundredFold = 4
End Enum

Sub Despace()
    Dim strTemp As String
    Dim sngStart As Single
    Dim dblStart As Double
    Dim lngStart As Long
    Dim oRE As Object
    Dim curFreq As Currency
    Dim curStart As Currency
    Dim curEnd As Currency
    Dim vItem As Variant
    Dim strFind As String
    Dim lngLoop As Long
    Dim vParsed As Variant
    Dim strWords() As String
    Dim colWords As New Collection
    Dim dicWords As Object
    Dim oMatches As Object
    Dim oM As Object
    Dim strFileData As String
    Dim strTestString As String
    Dim lngSize As Long
    Dim lngIterator As Long
    Dim lngPlainStuffed As Long
    Const cIterations As Long = 21
    Dim colLog As New Collection
    Dim lngFirstHit As Long
    Dim strCurrentTask As String
    Const cPath As String = "c:\users\mark\documents\"
    Dim clsXL As New clsWksFuncTrim

    vParsed = Array()
    
    Open cPath & "gettysburgaddress.txt" For Input As #1
    strFileData = Input(LOF(1), #1)
    Close
    '=======================================================
    'iterate the different codes with the following
    '   * first paragraph
    '   * entire file contents
    '   * x10 and x100 copies of the entire file contents
    'for each iteration,
    '   test with the base text (as written)
    '   test with inserted spaces.
    '=======================================================
    CPUFrequency curFreq
    For lngSize = 1 To 4
        strTestString = StringSizes(strFileData, lngSize)
        For lngPlainStuffed = 0 To 1
            If lngPlainStuffed = 1 Then
                strTestString = StuffWithSpaces(strTestString, 42)
            End If
            strCurrentTask = lngSize & vbTab & Array("Plain: ", "Stuffed: ")(lngPlainStuffed) & vbTab & Len(strTestString) & vbTab & InStr(strTestString, "  ")
            For lngIterator = 1 To cIterations
                CPUTickCount curStart
                lngFirstHit = InStr(strTestString, "zz")
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Max Instr() time: " & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = JustTwo(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "just two" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (2)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = ThreeTwo(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "three & two" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (3&2)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = MultiLengths(strTemp, Array(19, 11, 7, 3, 2))
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Multiple replaces" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Multi)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                Set oRE = CreateObject("vbscript.regexp")
                oRE.Global = True
                oRE.Pattern = "  +"
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = RegexpReplace(strTemp, oRE)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "regexp '  +'" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Regexp 1)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
            
                oRE.Pattern = " {2,}"
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = RegexpReplace(strTemp, oRE)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "regexp ' {2,}'" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Regexp 2)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
            
                oRE.Pattern = " \s+"
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = RegexpReplace(strTemp, oRE)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "regexp ' \s+'" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Regexp 3)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = RegexpReplace1(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "RegexpReplace1" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (RegexpReplace1)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = RegexpReplace2(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "RegexpReplace2" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (RegexpReplace2)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = RegexpReplace3(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "RegexpReplace3" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (RegexpReplace3)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                If Len(strTemp) < 32768 Then
                    strTemp = strTestString
                    sngStart = Timer
                    lngStart = getTickCount()
                    CPUTickCount curStart
                    strTemp = clsXL.CleanInternalSpaces(strTemp)
                    CPUTickCount curEnd
                    colLog.Add strCurrentTask & vbTab & "clsXL trim" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                    If strTemp <> StringSizes(strFileData, lngSize) Then
                        Debug.Print "strTemp not cleaned properly (clsXL trim)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
    '                    Stop
                    End If
                    
                    strTemp = strTestString
                    sngStart = Timer
                    lngStart = getTickCount()
                    CPUTickCount curStart
                    strTemp = WksFunctionTrim(strTemp)
                    CPUTickCount curEnd
                    colLog.Add strCurrentTask & vbTab & "WksFunc Trim" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                    If strTemp <> StringSizes(strFileData, lngSize) Then
                        Debug.Print "strTemp not cleaned properly (WksFunc Trim)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
    '                    Stop
                    End If
                End If
                
                strTemp = strTestString
                Erase vParsed
                CPUTickCount curStart
                vParsed = Split(strTemp, " ")
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split time: " & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                
                strTemp = strTestString
                Erase vParsed
                CPUTickCount curStart
                vParsed = Split(strTemp, "  ")
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split2 time: " & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = SplitJoin(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split & Join" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split & Join)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strWords = Split(strTemp, " ")
                strTemp = vbNullString
                CPUTickCount curStart
                strTemp = Join(strWords, " ")
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Join time: " & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = Split2Join(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split2 & Join" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split2 & Join)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = SplitCol(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split & col" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split & col)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = Split2Col(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split2 & col" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split2 & col)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                    
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = SplitDic(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split & dic" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split & dic)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                    
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = Split2Dic(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split2 & dic" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split2 & dic)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = SplitConcat(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split & concat" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split & concat)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = Split2Concat(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split2 & concat" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split2 & concat)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = SplitBuffer(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split & buffer" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split & buffer)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = Split2Buffer(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split2 & buffer" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split2 & buffer)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
                
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = Split2BufferFcn(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "Split2BufferFcn" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Split2BufferFcn)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
            
'                Erase strWords
                oRE.Pattern = "[^ ]+"
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = RegexParse(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "regexp parse" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (Regexp parse)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
            
                strTemp = strTestString
                sngStart = Timer
                lngStart = getTickCount()
                CPUTickCount curStart
                strTemp = RegexParseBuffer(strTemp)
                CPUTickCount curEnd
                colLog.Add strCurrentTask & vbTab & "RegexParseBuffer" & vbTab & "CPU cycles: " & vbTab & Format((curEnd - curStart) / curFreq, "0.000000000")
                If strTemp <> StringSizes(strFileData, lngSize) Then
                    Debug.Print "strTemp not cleaned properly (RegexParseBuffer)." & vbTab & "lngSize: " & lngSize & vbTab & "lngPlainStuffed: " & lngPlainStuffed
'                    Stop
                End If
            
            Next lngIterator
            DoEvents
        Next lngPlainStuffed
    Next lngSize
    
    Open cPath & "DeSpaceLog.txt" For Output As #1
    For Each vItem In colLog
        Print #1, vItem
    Next
    Close
    Debug.Print Now, "Despace() Finished"
    AppActivate Application.Caption
    Set clsXL = Nothing
    MsgBox "Despace() Finished", vbOKOnly, Now
End Sub

Function JustTwo(ByVal parmString As String) As String
    '================================================
    'Replace all double space strings with a single space.
    'Iterate until there are no more double space character
    'strings
    '================================================
    Dim strTemp As String
    strTemp = parmString
    Do Until InStr(strTemp, "  ") = 0
        strTemp = Replace(strTemp, "  ", " ")
    Loop
    JustTwo = strTemp
End Function

Function MultiLengths(ByVal parmString As String, _
                    ByVal parmLengths As Variant) As String
    '================================================
    'Iterate the parmLengths array and invoke the Replace() function
    'with a space string of each length.
    '================================================
    Dim vItem As Variant
    Dim strTemp As String
    Dim strFind As String
    
    strTemp = parmString
    For Each vItem In parmLengths
        strFind = Space(vItem)    'create a vItem length string of spaces
        Do Until InStr(strTemp, strFind) = 0
            strTemp = Replace(strTemp, strFind, " ")
        Loop
    Next
    MultiLengths = strTemp
End Function

Function RegexParse(ByVal parmString As String) As String
    '================================================
    'Use local static regexp object to parse the words.
    'Copy the parsed words to an array and Join them with
    'a single space delimiter
    '================================================
    Dim oMatches As Object
    Dim oM As Object
    Dim strWords() As String
    Dim lngLoop As Long
    Static oRE As Object
    
    If oRE Is Nothing Then
        Set oRE = CreateObject("vbscript.regexp")
        oRE.Global = True
        oRE.Pattern = "[^ ]+"
    End If
        
    Set oMatches = oRE.Execute(parmString)
    ReDim strWords(0 To oMatches.Count - 1)
    lngLoop = 0
    For Each oM In oMatches
        strWords(lngLoop) = oM.Value
        lngLoop = lngLoop + 1
    Next
    RegexParse = Join(strWords, " ")
End Function

Function RegexParseBuffer(ByVal parmString As String) As String
    '================================================
    'Use local static regexp object to parse the words.
    'Copy the parsed words to the buffer
    '================================================
    Dim oMatches As Object
    Dim oM As Object
    Static oRE As Object
    Dim strBuffer As String
    Dim lngWordPosn As Long
    Dim strTemp As String
    
    If oRE Is Nothing Then
        Set oRE = CreateObject("vbscript.regexp")
        oRE.Global = True
        oRE.Pattern = "[^ ]+"
    End If
        
    Set oMatches = oRE.Execute(parmString)
    strBuffer = Space(Len(parmString))
    
    lngWordPosn = 1
    For Each oM In oMatches
        strTemp = oM.Value
        Mid$(strBuffer, lngWordPosn, Len(strTemp)) = strTemp
        lngWordPosn = lngWordPosn + Len(strTemp) + 1
    Next
    RegexParseBuffer = RTrim(strBuffer)
End Function

Function RegexpReplace(ByVal parmString As String, parmRegexp As Object) As String
    '================================================
    'Use parameter regexp object to remove duplicate spaces.
    'The parameter regexp object will already have its pattern property set
    'by the calling code.
    '================================================
    RegexpReplace = parmRegexp.Replace(parmString, " ")
End Function

Function RegexpReplace1(ByVal parmString As String) As String
    '================================================
    'Use local static regexp object to remove duplicate spaces
    '================================================
    Static oRE As Object
    If oRE Is Nothing Then
        Set oRE = CreateObject("vbscript.regexp")
        oRE.Global = True
        oRE.Pattern = "  +"
    End If
    RegexpReplace1 = oRE.Replace(parmString, " ")
End Function

Function RegexpReplace2(ByVal parmString As String) As String
    '================================================
    'Use local static regexp object to remove duplicate spaces
    '================================================
    Static oRE As Object
    If oRE Is Nothing Then
        Set oRE = CreateObject("vbscript.regexp")
        oRE.Global = True
        oRE.Pattern = " {2,}"
    End If
    RegexpReplace2 = oRE.Replace(parmString, " ")
End Function

Function RegexpReplace3(ByVal parmString As String) As String
    '================================================
    'Use local static regexp object to remove duplicate spaces
    '
    'WARNING: This pattern will remove characters other than
    '       space characters due to the use of the \s in the pattern
    '================================================
    Static oRE As Object
    If oRE Is Nothing Then
        Set oRE = CreateObject("vbscript.regexp")
        oRE.Global = True
        oRE.Pattern = " \s+"
    End If
    RegexpReplace3 = oRE.Replace(parmString, " ")
End Function

Function SplitBuffer(ByVal parmString As String) As String
    '================================================
    'Split() string with single space character delimiter,
    'assign non-empty strings to next output buffer position,
    'returned the trimmed output buffer string
    '================================================
    Dim strParsed() As String
    Dim strTemp As String
    Dim lngLoop As Long
    Dim lngWordPosn As Long
    Dim strBuffer As String
    
    strParsed = Split(parmString, " ")
    strTemp = vbNullString
    
    lngWordPosn = 1
    strBuffer = Space(Len(parmString))
    For lngLoop = 0 To UBound(strParsed)
        strTemp = strParsed(lngLoop)
        If Len(strTemp) <> 0 Then
            Mid$(strBuffer, lngWordPosn) = strTemp
            lngWordPosn = lngWordPosn + Len(strTemp) + 1
        End If
    Next
    SplitBuffer = RTrim(strBuffer)

End Function

Function Split2Buffer(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitBuffer, except using a double space
    'delimiter for the Split() function.
    '
    'Split() string with a double space character delimiter,
    'assign non-empty strings to next output buffer position,
    'returned the trimmed output buffer string
    '================================================
    Dim strParsed() As String
    Dim strTemp As String
    Dim lngLoop As Long
    Dim lngWordPosn As Long
    Dim strBuffer As String
    
    strParsed = Split(parmString, "  ")
    strTemp = vbNullString
    
    lngWordPosn = 1
    strBuffer = Space(Len(parmString))
    For lngLoop = 0 To UBound(strParsed)
        strTemp = Trim(strParsed(lngLoop))
        If Len(strTemp) <> 0 Then
            Mid$(strBuffer, lngWordPosn, Len(strTemp)) = strTemp
            lngWordPosn = lngWordPosn + Len(strTemp) + 1
        End If
    Next
    Split2Buffer = RTrim(strBuffer)

End Function

Function Split2BufferFcn(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitBuffer, except using a double space
    'delimiter for the Split() function.
    '
    'Split() string with a double space character delimiter,
    'assign non-empty strings to next output buffer position,
    'returned the trimmed output buffer string
    '================================================
    Dim strParsed() As String
    Dim strTemp As String
    Dim lngLoop As Long
    Dim lngWordPosn As Long
    
    strParsed = Split(parmString, "  ")
    strTemp = vbNullString
    
    lngWordPosn = 1
    Split2BufferFcn = Space(Len(parmString))
    For lngLoop = 0 To UBound(strParsed)
        strTemp = Trim(strParsed(lngLoop))
        If Len(strTemp) <> 0 Then
            Mid$(Split2BufferFcn, lngWordPosn) = strTemp
            lngWordPosn = lngWordPosn + Len(strTemp) + 1
        End If
    Next
    Split2BufferFcn = RTrim(Split2BufferFcn)

End Function

Function SplitCol(ByVal parmString As String) As String
    '================================================
    'Split() string with single space character delimiter,
    'adding the non-empty strings to a collection object.
    'Copy the collection items to an array and
    'Join() them as the returned value
    '================================================
    Dim strParsed() As String
    Dim strTemp As String
    Dim lngLoop As Long
    Dim strWords() As String
    Dim colWords As New Collection
    Dim vItem As Variant
    
    strParsed = Split(parmString, " ")
    For lngLoop = 0 To UBound(strParsed)
        strTemp = strParsed(lngLoop)
        If Len(strTemp) <> 0 Then
            colWords.Add strTemp
        End If
    Next
    ReDim strWords(1 To colWords.Count)
    lngLoop = 1
    For Each vItem In colWords
        strWords(lngLoop) = vItem
        lngLoop = lngLoop + 1
    Next
    SplitCol = Join(strWords, " ")

End Function

Function Split2Col(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitCol, except using a double space
    'delimiter for the Split() function.
    '
    'Split() string with double space character delimiter,
    'adding the non-empty strings to a collection object.
    'Copy the collection items to an array and
    'Join() them as the returned value
    '================================================
    Dim strParsed() As String
    Dim strTemp As String
    Dim lngLoop As Long
    Dim strWords() As String
    Dim colWords As New Collection
    Dim vItem As Variant
    
    strParsed = Split(parmString, "  ")
    For lngLoop = 0 To UBound(strParsed)
        strTemp = Trim(strParsed(lngLoop))
        If Len(strTemp) <> 0 Then
            colWords.Add strTemp
        End If
    Next
    ReDim strWords(1 To colWords.Count)
    lngLoop = 1
    For Each vItem In colWords
        strWords(lngLoop) = vItem
        lngLoop = lngLoop + 1
    Next
    Split2Col = Join(strWords, " ")

End Function

Function SplitConcat(ByVal parmString As String) As String
    '================================================
    'Split() string with single space character delimiter,
    'concatenate non-empty strings with a trailing space character
    '================================================
    Dim strParsed() As String
    Dim strTemp As String
    Dim lngLoop As Long
    Dim strSplitConcat As String
    
    strParsed = Split(parmString, " ")
    lngLoop = 0
    strTemp = vbNullString
    
    For lngLoop = 0 To UBound(strParsed)
        strTemp = strParsed(lngLoop)
        If Len(strTemp) <> 0 Then
            strSplitConcat = strSplitConcat & strTemp & " "     'concatenate with space
        End If
    Next
    SplitConcat = RTrim(strSplitConcat)

End Function

Function Split2Concat(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitConcat, but using double space character
    'string as delimiter for the Split() function
    '
    'Split() string with a double space character delimiter,
    'concatenate non-empty strings with a trailing space character
    '================================================
    Dim strParsed() As String
    Dim strTemp As String
    Dim lngLoop As Long
    Dim strSplit2Concat As String
    
    strParsed = Split(parmString, "  ")
    strTemp = vbNullString
    
    For lngLoop = 0 To UBound(strParsed)
        strTemp = Trim(strParsed(lngLoop))
        If Len(strTemp) <> 0 Then
            strSplit2Concat = strSplit2Concat & strTemp & " "       'concatenate with space
        End If
    Next
    Split2Concat = RTrim(strSplit2Concat)       'remove trailing space

End Function

Function SplitDic(ByVal parmString As String) As String
    '================================================
    'Split() string with a single space character delimiter,
    'adding non-empty strings to the dictionary,
    'then Join() the dictionary object's items array.
    '================================================
    Dim strParsed() As String
    Dim lngLoop As Long
    Dim lngKey As Long
    Dim strTemp As String
    
    Static dicWords As Object
    If dicWords Is Nothing Then
        Set dicWords = CreateObject("scripting.dictionary")
    Else
        dicWords.RemoveAll
    End If
    strParsed = Split(parmString, " ")
    lngKey = 1
    
    For lngLoop = 0 To UBound(strParsed)
        strTemp = strParsed(lngLoop)
        If Len(strTemp) <> 0 Then
            dicWords.Add CStr(lngKey), strTemp
            lngKey = lngKey + 1
        End If
    Next
    SplitDic = Join(dicWords.items, " ")

End Function

Function Split2Dic(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitDic, but using double space character
    'string as delimiter for the Split() function
    '
    'Split() string with a double space character delimiter,
    'adding non-empty strings to the dictionary,
    'then Join() the dictionary object's items array
    '================================================
    Dim strParsed() As String
    Dim lngLoop As Long
    Dim lngKey As Long
    Dim strTemp As String
    
    Static dicWords As Object
    If dicWords Is Nothing Then
        Set dicWords = CreateObject("scripting.dictionary")
    Else
        dicWords.RemoveAll
    End If
    strParsed = Split(parmString, "  ")
    lngKey = 1
    
    For lngLoop = 0 To UBound(strParsed)
        strTemp = Trim(strParsed(lngLoop))
        If Len(strTemp) <> 0 Then
            dicWords.Add CStr(lngKey), strTemp
            lngKey = lngKey + 1
        End If
    Next
    Split2Dic = Join(dicWords.items, " ")

End Function

Function SplitJoin(ByVal parmString As String) As String
    '================================================
    'Split() string with single space character delimiter,
    'move non-empty strings to the front of the strParsed array,
    'Redim the strParsed array down to the number of words we have,
    'then Join() the strParsed array items with
    'a single space character
    '================================================
    Dim strParsed() As String
    Dim lngLoop As Long
    Dim lngWord As Long
    Dim strTemp As String
    
    strParsed = Split(parmString, " ")
    lngWord = 0
    
    'Move non-empty strings to the front of the strParsed array
    For lngLoop = 0 To UBound(strParsed)
        strTemp = strParsed(lngLoop)
        If Len(strTemp) <> 0 Then
            strParsed(lngWord) = strTemp
            lngWord = lngWord + 1
        End If
    Next

    'reduce size of the strParsed array to equal the number
    'of non-empty strings we found.
    ReDim Preserve strParsed(0 To lngWord - 1)
    SplitJoin = Join(strParsed, " ")

End Function

Function SplitJoin_InPlace(ByVal parmString As String) As String
    '================================================
    'Split() string with single space character delimiter,
    'move non-empty strings to the front of the strParsed array,
    'Redim the strParsed array down to the number of words we have,
    'then Join() the strParsed array items with
    'a single space character
    '================================================
    Dim strParsed() As String
    Dim lngLoop As Long
    Dim lngWord As Long
    Dim strTemp As String
    
    strParsed = Split(parmString, "  ")
    lngWord = 0
    
    'Move non-empty strings to the front of the strParsed array
    For lngLoop = 0 To UBound(strParsed)
        strTemp = strParsed(lngLoop)
        If Len(strTemp) <> 0 Then
            strParsed(lngWord) = strTemp
            lngWord = lngWord + 1
        End If
    Next

    'reduce size of the strParsed array to equal the number
    'of non-empty strings we found.
    ReDim Preserve strParsed(0 To lngWord - 1)
    SplitJoin_InPlace = Join(strParsed, " ")

End Function

Function Split2Join(ByVal parmString As String) As String
    '================================================
    'Mostly the same as SplitJoin, but using double space character
    'string as delimiter for the Split() function
    '
    'Split() string with single space character delimiter,
    'move non-empty strings to the front of the strParsed array,
    'Redim the strParsed array down to the number of words we have,
    'then Join() the strParsed array items with
    'a single space character
    '================================================
    Dim strParsed() As String
    Dim lngLoop As Long
    Dim lngWord As Long
    Dim strTemp As String
    
    strParsed = Split(parmString, "  ")
    lngWord = 0
    
    'Move non-empty strings to the front of the strParsed array
    For lngLoop = 0 To UBound(strParsed)
        strTemp = Trim(strParsed(lngLoop))
        If Len(strTemp) <> 0 Then
            strParsed(lngWord) = strTemp
            lngWord = lngWord + 1
        End If
    Next

    'reduce size of the strParsed array to equal the number
    'of non-empty strings we found.
    ReDim Preserve strParsed(0 To lngWord - 1)
    Split2Join = Join(strParsed, " ")

End Function

Function StringSizes(ByVal parmString As String, parmSizeRequest As eSizeRequest) As String
    '================================================
    'Return size permutation of Gettysburg address.
    'Parameters:
    '   1: First paragraph
    '   2: The (original) address = parmString
    '   3: 10 concatenations of the address
    '   4: 100 concatenations of the address
    '================================================
    Dim lngLoop As Long
    Dim strTemp() As String
    Select Case parmSizeRequest
        
        Case eSizeRequest.eFirstParagraph   'first paragraph
            StringSizes = Split(parmString, vbCrLf, 2)(0)
            
        Case eSizeRequest.eSameAsDocument   'same as parameter
            StringSizes = parmString
        
        Case eSizeRequest.eTenFold          'repeat ten times
            ReDim strTemp(1 To 10)
            For lngLoop = 1 To 10
                strTemp(lngLoop) = parmString
            Next
            StringSizes = Join(strTemp, vbNullString)
        
        Case eSizeRequest.eHundredFold      'repeat one hundred times
            ReDim strTemp(1 To 100)
            For lngLoop = 1 To 100
                strTemp(lngLoop) = parmString
            Next
            StringSizes = Join(strTemp, vbNullString)
        
        Case Else
            StringSizes = vbNullString
            
    End Select
    
End Function

Function StuffWithSpaces(ByVal parmString As String, parmSeed) As String
    '================================================
    'Add Random number of internal space characters
    '================================================
    Dim lngRnd As Long
    Dim strWords() As String
    Dim lngLoop As Long
    Const cMaxSpaces As Long = 26
    Dim lngSum As Long      'used to verify avg inserter spaces length
    
    Rnd -1                  'reset random sequence
    Randomize parmSeed      'begin random sequence
    strWords = Split(parmString, " ")
    For lngLoop = 0 To UBound(strWords) - 1
        lngRnd = Int(Rnd() * cMaxSpaces) + 1
        strWords(lngLoop) = strWords(lngLoop) & Space(lngRnd)
    Next
    StuffWithSpaces = Join(strWords, vbNullString)
End Function

Sub testit()
    'minimize code window before invoking test code
    Debug.Print Now, "Before Despace"
    SendKeys "% N", False
    DoEvents
    Despace
    Debug.Print Now, "After Despace"
End Sub

Function ThreeTwo(ByVal parmString As String) As String
    '================================================
    'Replace all three consecutive spaces with one space,
    'then replace all two consecutive spaces with one space
    '================================================
    Dim strTemp As String
    strTemp = parmString

    'Replace three space strings with a single space until
    'no more instances of three space strings exist
    Do Until InStr(strTemp, "   ") = 0
        strTemp = Replace(strTemp, "   ", " ")
    Loop

    'Replace two space strings with a single space until no
    'more instances of two space strings exist
    Do Until InStr(strTemp, "  ") = 0
        strTemp = Replace(strTemp, "  ", " ")
    Loop
    ThreeTwo = strTemp
End Function

Function WksFunctionTrim(ByVal parmString As String) As String
    '================================================
    'If running in the Excel VBA environment, invoke the
    'Trim Worksheetfunction
    '================================================
    WksFunctionTrim = WorksheetFunction.Trim(parmString)
End Function
```
 <h2>References and Related Articles</h2>
 <p>Fast String Builder class - <a href="http%3AA_8311.html" rel="nofollow">http:A_8311.html</a></p>
 <p>Better Concat Function - &nbsp;<a href="http%3AA_7811.html" rel="nofollow">http:A_7811.html</a></p>
 <p>Using Regular Expressions in VBA environment - &nbsp;<a href="http%3AA_1336.html" rel="nofollow">http:A_1336.html</a></p>
 <p>Analysis of the VB's Random Number Generator Functions - &nbsp;<a href="http%3AA_11114.html" rel="nofollow">http:A_11114.html</a></p>
