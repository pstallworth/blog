Title: Online Room Swaps - For Students
Author: Paul Stallworth
Category: Odyssey
Tags: cbord odyssey hms room swap student
Date: 2013-3-6
Summary: Creating a custom room selection page for use within Odyssey Web

Updated: 2013-10-18

<link rel="stylesheet" href="./static/css/custom.css">
We having been using the SelectRoom function to allow students to change
rooms or change buildings for a few years now. It works really well for
allowing students to change their assignment for an upcoming semester,
and we even drop in customizations to allow for mid-semester room
changes as needed. The problem has always been when two students want to
make a 1:1 swap with their spaces. Previously we had no way to allow
this online so we would a) have them both come to our office (together)
and request the swap or b) have them email us acknowledging their desire
to swap with the other student. This was cumbersome for the students and
our staff, especially when multiple students wanted to swap amongst
suitemates and we were waiting for 8 email confirmations and the
logistics of the swap.

I decided to build a custom page that would allow students to swap rooms
online without the need for our staff to be involved. We already use
*StudentRequest* functions with forms that submit to custom pages, so I
chose to use a *StudentRequest* function for this as well.  Setting up
the StudentRequest is pretty simple, just insert a new StudentRequest
function for the desired term, and stick a form in the *Custom Content
Code* section of the *Request Detail* page that submits to your custom
ASP.

![Custom Content](static/images/CustomContent1.png "Custom Content Page")

Here's the code in the block:
```Asp
Response.write "<script type=""text/javascript"" src=""script/validateSwapInput.js""></script>" & vbNewLine

FunctionKey = Request.QueryString("Function")

Response.Write "<form action=""RoomSwapRequest.asp?Function=" & FunctionKey & """ method=""POST"" id=""frmRequestDetail2"">" & vbCrLf
Response.Write "<input type=""text"" name=""swapid"" id=""swapid"" size=""9"" maxlength=""8"">"
Response.write "<input type=""hidden"" name=""Save"" id=""Save"" value=""1"" />"
Response.write "<br />"
Response.write "<input type=""submit"" name=""btnSave2"" id=""btnSave2"" value=""Submit Request"" /></form>" & vbCrLf
Response.Write "<form action=""RoomSwapRequest.asp?Function=" & FunctionKey & """ method=""POST"" id=""frmDoNotAgree"">" & vbCrLf
Response.Write "<input type=""submit"" value=""Clear Swap Request"" id=""btnClearRequest"" />" & vbCrLf
Response.Write "<input type=""hidden"" id=""ClearSwapRequest"" name=""ClearSwapRequest"" value=""1"" /></form>" & vbCrLf

Response.Write "</div></div>" & vbCrLf
Response.Write "<div id=""footer"">" & vbCrLf & Footer() & vbCrLf & "</div>" & vbCrLf
Response.Write "</body></html>" & vbCrLf
Set oPage = Nothing
Set oStudent = Nothing

Response.End
```
Next I needed to figure out the flow of how the swap would be performed.
I decided the easiest way would be that the two students must log on and mutually
request to swap with one another.  The order shouldn't matter, so long as
they both submit a request.  Instead of using a table in the database, I
created a new attribute called Swap Request.  This attribute would be used to store the ID number of the
student they wanted to swap with, and it would not be visible to anyone
except our staff (to avoid FERPA issues).  As each student logs on and
requests to swap, the ID number is recorded in their attribute and the system basically waits
for the other student to make the reciprocal request, at which point the swap is performed. 
To perform the actions with attribute we'll need a handle to both PatronWrite and PatronRead,
and the attribute ID number. The other parameters we get for free as
a part of the application. Let's stub the page real quick, write the ID
number from the form, and read that student's Swap Request attribute:
```Asp
	!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
		"http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
	<html xmlns="http://www.w3.org/1999/xhtml">
	<head>
	<!--#include file="include/StudentCacheAll.inc" -->
	<!--#include file="include/AuthenticatePlugin.inc" -->
	<!--#include file="include/EmailPlugin.inc" -->
	</head>
	<body>
	<%
	On Error Resume Next

	Call PageAuthenticate()
	Call UpdateSessionActivity()

	Const ATTRIB_SWAP_REQUEST = 3761

	Dim oStudent, oFunction, oTerm, swapID

	FunctionKey = Request.QueryString("Function")
	Set oFunction = New CFunction
	oFunction.Initialize FunctionKey
	Set oTerm = New CTerm
	oTerm.Initialize oFunction.TermKey
	Set oStudent = New CStudent
	oStudent.Initialize PatronID

	swapID = Request.Form("swapid") 'swapID is ID entered in input box'

	Set oPatronWrite = GetClass("HMSDBSrv.PatronWrite")
	bSuccess = oPatronWrite.InsertAttributeValue(OdysseyToken(), oStudent.Key, ATTRIB_SWAP_REQUEST, oTerm.StartDate, oTerm.EndDate, swapID, vFailedRows)

	Set oPatronRead = GetClass("HMSDBSrv.PatronRead")
	Set rs = oPatronRead.GetPatron(OdysseyToken(), ,swapID)

	swapKey = rs("Patron_Key")

	Set oAttributeRead = GetClass("HMSDBSrv.AttributeRead")
	Set rs = oAttributeRead.BrowsePatronAttributesAtDate(OdysseyToken(), swapKey, oTerm.StartDate)

	Do Until rs("Attribute_ID") = ATTRIB_SWAP_REQUEST 'attribute id for the Swap Request' 
		rs.MoveNext
	Loop

	swapStudentSwapRequest = rs("Value")

	%>
	</body>
	</html>
```

Nothing too fancy here, and half of it is boilerplate. The first few
lines are page setup that gets us access to the web function, the term
the function is associated with, and the student logged in submitting
the request. The *InsertAttributeValue* call sets the attribute value
for the dates given, in this case the start and end date of the
semester. Then we use the submitted ID to get the Patron_Key, which is
required in the call to *BrowsePatronAttributesAtDate*. From there we
iterate through the results until we come to the record we want and put
the value in our variable for later.

So far so good, now how do we swap them?

**Going to a swap meet**
------------------------

I knew that the staff web page had an option for performing a swap, so I
looked there for inspiration. In *PatronChangeRoomResults.asp* I found
the call to *ChangeAssignment3* that CBORD uses when you opt to do a
room swap. I looked it up in the Web API and decided it was what I
needed as well. As of this writing *ChangeAssignment3* is deprecated and
should be updated to *ChangeAssignment5*, but I haven’t gotten around to
that yet. Doing the swap at this point involves verifying the mutual
request for the swap, getting each student’s assignment, and calling
*ChangeAssigment3* to swap them. The definition for *ChangeAssignment3*
looks like this:

	ChangeAssignment3(Token,
		ChangeDate : Date,
		ElementKey : String,
		ToFacilityKey : Long,
		ByRef FailedRows : Variant,
		Optional ByRef NewElementKey : Long,
		Optional ByRef NewSwapElementKey : Long,
		Optional UseOverlow : Boolean = False,
		Optional MarkReplacedAsDeleted : Boolean = True,
		Optional ReplaceCurrent : Boolean = False,
		Optional SwapElementKey : String,
		Optional SpaceLinkFacilityKey : Long,
		Optional SwapSpaceLinkFacilityKey : Long,
	) : Boolean

And CBORD calls it like this in the web staff:

	bSuccess =
	oContractWrite.ChangeAssignment3(GetUserSession("StaffToken"),
		dChange,lElementKey, lFacilityToKey, vFailed, , , bOverflow, , , lElementSwapKey)


In our case we will never use overflow spaces, so we can disregard
bOverflow, which means the pieces we need are the token of the logged in
student, the date to make the change, the contract element key for the
student logged in, the facility key of the student to swap with (where
the logged in student is going), a variant for failures, and the
contract element of the student we are swapping with. The only caveat is
the optional **MarkReplaceAsDeleted** which says that replaced assignments
are set to a cancelled state if True, and are deleted from the database
if False. I don’t want to have a cancelled record hanging around after
each swap, so I mark this False when I call it.

	bAssignSuccess = oContractWrite.ChangeAssignment3(OdysseyToken(),
		dAssignStart ,lElementKey, lFacilityToKey, vFailedRows2, , , ,False , , lElementSwapKey)

So how do we get all those required pieces? The Token and ChangeDate are
easy since they are part of the Student and Term objects, respectively.
We need to call *GetContractElements* on both students to get access to
the contract element and facility key, which looks like this.

	Set oContractRead = GetClass("HMSDBSrv.ContractRead")
	Set rsElements = oContractRead.GetContractElements(OdysseyToken(), oStudent.Key, oTerm.StartDate, oTerm.EndDate)
			
	Do Until rsElements.EOF
			 If Not IsNull(rsElements("Facility_Key")) Then
			If (rsElements("State_ID") = 1) Then 'This is the preliminary assignment
				lElementKey = rsElements("Element_Key") 
				lFacilityKey = rsElements("Facility_Key")
			End If
		End If
		rsElements.MoveNext
	Loop

Then we do the same thing for the student that we are swapping with:

```
Set rsElements = oContractRead.GetContractElements(OdysseyToken(), swapKey, oTerm.StartDate, oTerm.EndDate) 
Do Until rsElements.EOF
	If Not IsNull(rsElements("Facility_Key")) Then
		If (rsElements("State_ID") = 1) Then 'This is the preliminary assignment
			lFacilityToKey = rsElements("Facility_Key")
			lElementSwapKey = rsElements("Element_Key")
		End If
	End If
	rsElements.MoveNext
Loop
```

Now we have all the required pieces and can make the swap call:

	bAssignSuccess = oContractWrite.ChangeAssignment3(OdysseyToken(),
	dAssignStart ,lElementKey, lFacilityToKey, vFailedRows2, , , ,False , , lElementSwapKey)

bAssignSuccess will be true if the swap was successful, and false
otherwise, at which point we can inspect vFailedRows2 to see what
happened. Putting it all together we should have something like this:
  
```Asp
	!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
		"http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
	<html xmlns="http://www.w3.org/1999/xhtml">
	<head>
	<!--#include file="include/StudentCacheAll.inc" -->
	<!--#include file="include/AuthenticatePlugin.inc" -->
	<!--#include file="include/EmailPlugin.inc" -->
	</head>
	<body>
	<%
	On Error Resume Next

	Call PageAuthenticate()
	Call UpdateSessionActivity()

	Const ATTRIB_SWAP_REQUEST = 3761

	Dim oStudent, oFunction, oTerm, swapID

	FunctionKey = Request.QueryString("Function")
	Set oFunction = New CFunction
	oFunction.Initialize FunctionKey
	Set oTerm = New CTerm
	oTerm.Initialize oFunction.TermKey
	Set oStudent = New CStudent
	oStudent.Initialize PatronID

	swapID = Request.Form("swapid") 'swapID is ID entered in input box'

	Set oPatronWrite = GetClass("HMSDBSrv.PatronWrite")
	bSuccess = oPatronWrite.InsertAttributeValue(OdysseyToken(), oStudent.Key, ATTRIB_SWAP_REQUEST, oTerm.StartDate, oTerm.EndDate, swapID, vFailedRows)

	Set oPatronRead = GetClass("HMSDBSrv.PatronRead")
	Set rs = oPatronRead.GetPatron(OdysseyToken(), ,swapID)

	swapKey = rs("Patron_Key")

	Set oAttributeRead = GetClass("HMSDBSrv.AttributeRead")
	Set rs = oAttributeRead.BrowsePatronAttributesAtDate(OdysseyToken(), swapKey, 
	oTerm.StartDate)

	Do Until rs("Attribute_ID") = ATTRIB_SWAP_REQUEST 'attribute id for the Swap Request'
		rs.MoveNext
	Loop

	swapStudentSwapRequest = rs("Value")

	If oStudent.ID = swapStudentSwapRequest Then 'both students have requested each other'

	Set oContractRead = GetClass("HMSDBSrv.ContractRead")
	Set rsElements = oContractRead.GetContractElements(OdysseyToken(), oStudent.Key, oTerm.StartDate, oTerm.EndDate)
				
			 Do Until rsElements.EOF
		If Not IsNull(rsElements("Facility_Key")) Then
				If (rsElements("State_ID") = 1) Then 'This is the preliminary assignment'
					lElementKey = rsElements("Element_Key") 
					lFacilityKey = rsElements("Facility_Key")
				End If
			End If
			rsElements.MoveNext
		Loop

		Set rsElements = oContractRead.GetContractElements(OdysseyToken(), swapKey, oTerm.StartDate, oTerm.EndDate) 
		Do Until rsElements.EOF
			If Not IsNull(rsElements("Facility_Key")) Then
				If (rsElements("State_ID") = 1) Then 'This is the preliminary assignment'
					lFacilityToKey = rsElements("Facility_Key")
					lElementSwapKey = rsElements("Element_Key")
				End If
			End If
			rsElements.MoveNext
		Loop

	End If

	bAssignSuccess = oContractWrite.ChangeAssignment3(OdysseyToken(), dAssignStart ,lElementKey, lFacilityToKey, vFailedRows2, , , ,False , , lElementSwapKey)

	%>
	</body>
	</html>
```
One thing that has been absent through this discussion is error
checking and handling, and I did it on purpose so we could focus on the
core functions of the room swap. With that being said, there should be
some kind of error checking after each API call with at least a simple
 `Response.Redirect “Error.asp?Msg=Error”` to halt the processing. This
should be enough to get you started with building your own Room Swap
application, and in the [following posts](|filename|./RoomSwaps2.md) I’ll add additional conditions
to the swap, and go over a special case for swapping after move in.
