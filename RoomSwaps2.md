Title: Online Room Swaps - For Students - Part 2
Author: Paul Stallworth
Category: Odyssey
Tags: cbord odyssey hms room swap student
Date: 2013-3-13
Summary: Creating a custom room selection page for use within Odyssey Web

<link rel="stylesheet" href="./static/css/custom.css">
In the [last post](|filename|./RoomSwaps1.md) we went over building the basic page to allow students to swap rooms amongst themselves.  At this point the code functions properly and the swaps take place, but there are several issues that remain.  
  
These issues include:

1.  Not clearing the attribute means either student can undo the change.  
2.  No restrictions are made regarding which students may swap, so the only protections are institutional settings.  
3.  No restrictions are made regarding which facilities are available for swapping.  

Clearing the swap request
-
Handling #1 is not that difficult.  Since we use an attribute to track the requests, all we need to do is clear each student's attribute upon successful swap.  We get the result of the swap from the call to *ChangeAssignment3*, so we can wrap this in a test so that it only happens on a successful swap.

```Asp
If bAssignSuccess = True Then
    Dim bClearSuccess, bClearSuccess2
    Set oPatronWrite = GetClass("HMSDBSrv.PatronWrite")
    
    bClearSuccess = oPatronWrite.InsertAttributeValue(OdysseyToken(), oStudent.Key, ATTRIB_SWAP_REQUEST, oTerm.StartDate, oTerm.EndDate, "", vFailedRows3)
    
    bClearSuccess2 = oPatronWrite.InsertAttributeValue(OdysseyToken(), swapKey, ATTRIB_SWAP_REQUEST, oTerm.StartDate, oTerm.EndDate, "", vFailedRows3)
```

Restricting facilities
-
Skipping to #3 we will perform checks against facility attributes based upon the rules we setup for our swaps.  We will use Asset Types and Facility Attributes to control what rooms are allowed to swap.

We already have a handle to the contract elements in rsElements (from previous post) so we can add a couple more lines to get some additional fields to check:

```  
sAssetType = rsElements("AssetTypeName")
sFacilityName = rsElements("Name")
```  

Next we'll use the facility key (again from before) to pull facility attributes for the room the first student is assigned to:

```  
Set rs = oAttributeRead.BrowseFacilityAttributesAtDate(OdysseyToken(), lFacilityKey, oTerm.StartDate)
Dim sSpecialUse, sStudentFacilityType, sSwapStudentFacilityType
sSpecialUse = "No"
    			
Do Until rs.EOF
    If Not IsNull(rs("FacilityAttribute_Key")) Then
	    If (rs("Name") = "Special Use") Then
		    sSpecialUse = rs("Value")
	    ElseIf (rs("Name") = "Student Facility Type") Then
		    sStudentFacilityType = rs("Value")
	    End IF
    End IF
    rs.MoveNext
Loop
```  
_Note about our business process: We use the Special Use attribute as a catch all for VIP rooms, and other spaces we do not want in our assignment or room selection process.  The Student Facility Type attribute indicates first-year and non-first-year facilities.  We also track certain room types with Asset Types such as ADA rooms and athletic spaces._  

CBORD uses a common redirect to Error.asp for handling errors, and we do the same here.

```
IF sSpecialUse = "Yes" Then
    Response.Redirect "Error.asp?Msg=Room not eligible for swap"
ElseIf sAssetType = "CA" OR sAssetType = "LJ" OR sAssetType = "PC" THEN
    Response.Redirect "Error.asp?Msg=Room unavailable for swap (asset type)"
END IF
```

We pull the same information for the second student, this time using lFacilityToKey, as well as a check to make sure the facility types match (to prevent first-year and non-first-year students from swapping rooms).

```
If sSwapStudentFacilityType <> sStudentFacilityType THEN
    Response.Redirect "Error.asp?Msg=Cannot swap with student in that facility."
ElseIf sSpecialUse = "Yes" Then
	Response.Redirect "Error.asp?Msg=Room not eligible for swap"
ElseIf sAssetType = "CA" OR sAssetType = "LJ" OR sAssetType = "PC" THEN
	Response.Redirect "Error.asp?Msg=Room unavailable for swap (asset type)"
End if

```

