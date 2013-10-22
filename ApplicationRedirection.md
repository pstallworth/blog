Title: Redirecting Application Flow in Odyssey Web Functions
Author: Paul Stallworth
Category: Odyssey
Tags: cbord odyssey hms application
Date: 2013-10-18
Summary: How to skip pages in the Odyssey web application process

<link rel="stylesheet" href="./static/css/custom.css">

Business case:  When students submit an online cancellation we ask a few 
questions about their reason for cancelling to give us a better idea of 
how to process the request, as well as provide information for future 
analysis.  One of the issues we were running into was that we had a separate 
off-campus permit request application that was not part of our cancellation 
function.  Therefore, students could submit their cancellation, indicate that 
they were applying to live off-campus, but not take the second step and submit 
the off-campus permit request.

We decided to insert the off-campus permit request into the middle of the 
cancellation request, but only show the page based upon their selection from 
the previous page.  If their reason was 'Applying for an off-campus permit' 
we would redirect to the permit request page, otherwise we would skip it and 
take them to the signature page.

This screenshot of the web function highlights where each page lives and the flow.

--insert screenshot--

Getting User Input
------------------

Next we setup *ApplyPref1* with the fields we want to capture from the user.  
In our case we are only capturing one attribute, the Cancellation Reason.  
No custom code is required on this page.  

Skipping The Page
-----------------

When we get to *ApplyPref2* this is where we will either skip the page based 
upon the value submitted on the previous page or show the page.  Since the 
attribute value will be written when the user submits the previous page we 
can simply read it in the header section and decide where to go next.  The 
following code is what we put in the Custom Header Code section that will 
either skip the page at the beginning of processing or simply fall through 
and display the rest of the content for the page.

```Asp
Dim oAttrRead, cancelReason, nextPageName, FunctionKey

Set oAttrRead = GetClass("HMSDBSrv.AttributeRead")
Set rs = oAttrRead.GetPatronAttributeValues(OdysseyToken(), oStudent.Key, oTerm.StartDate, oTerm.EndDate)

'You must change this to where you want to land after you skip
nextPageName = "ApplyPref3"

'This is required in the query string for the page you will land on
FunctionKey = Request.QueryString("Function")

rs.Filter = "Name='Cancel Reason'"

If rs.EOF Then
    Response.Redirect "Error.asp?Msg='This is very bad, could not read attributes.'"
End If

cancelReason = rs.Fields(.Value.)

'Either we take the if block and redirect to next page or we stay on current page
If cancelReason <> "Applying for an Off-campus permit" Then
    .student chose any other reason, skip this page, do not show permit page
    Response.redirect "ApplyPref.asp?Function=" & FunctionKey & "&PageName=" & nextPageName
End If

'else we are going to show the rest of the page

Set oAttrRead = Nothing
```

In this example we set *nextPageName* to 'ApplyPref3' and performed a Resonse.Redirect. 
You could set it to any ApplyPref page you want to jump to.  For us *ApplyPref3* 
is not displayed, nor are any of the pages after it until ApplySignature, so 
this effectively causes a redirection to the signature page.  Since the 
preference pages are all built from the ApplyPref.asp page you can use the 
query string parameters to specify which ApplyPref page to go to.  If you wish 
to jump directly to ApplySignature.asp you will have to change out the 
Response.Redirect line with something like this:

```Asp
Server.Transfer "ApplySignature.asp"
```

There are some differences between using Response.Redirect and Server.Transfer, 
but personally I think it is safer to use Response.Redirect to a preference 
page and allow the normal flow control to take over from there.  When sending 
users to ApplySignature.asp, CBORD almost always uses Server.Transfer, so if 
you need to jump directly to ApplySignature.asp I would use that method as well. 


