---
layout: post
title: SQL Generation & Templating
description: Using templating (macro) approach to some problems
tags: tech regex
---
# Introduction
I recently answered a question (http://e-e.com/questions/29100944) about updating rows in a table. This was a self-join scenario where some of the rows served as update data sources for other rows with matching keys.

Most of the proposed solutions involved using T-SQL to generate the Update statements or a stored proc.  I took a different approach which proved quite flexible and produced testable results quickly.  I used SQL as replacement pattern for a few Find/Replace operations in a text editor.  Since I am a fan of regular expressions, my text editor of choice is Notepad++.  Since many solutions involve iteration with your users, flexibility and iteration speed are important characteristics.  

## User Requirements
Initially, there were only two types of columns to consider, date and non-date.  Bit column types were added later.  There are nearly 350 columns in this table and only two columns (the Primary key column: ID, and the source/target indicator: SkipImport) are excluded from the updating process.

1. Update statement for non-date columns, replace if TGT is Null and SRC Not Null  
``` SQL  
Update TGT 
Set TGT.$1 = SRC.$1
From Employeestbl TGT inner join Employeestbl SRC  
	on TGT.[Email] = SRC.[Email] 
Where TGT.SkipImport = 0 And SRC.SkipImport = 1 
And TGT.$1 is null and SRC.$1 is not null;```  

2. Update statement for date columns, replace if TGT is < SRC and prior condition  
``` SQL  
Update TGT 
Set TGT.$1 = SRC.$1
From Employeestbl TGT inner join Employeestbl SRC  
	on TGT.[Email] = SRC.[Email] 
Where TGT.SkipImport = 0 And SRC.SkipImport = 1 
And ((TGT.$1 is null And SRC.$1 is not null) 
	OR (TGT.$1 is not null And SRC.$1 is not null And TGT.$1 < SRC.$1)
	);```  

3. Update statement for bit columns, update if:  
	Sender is 1 and receiver is either 0 or null,  
	then field should be updated to 1.  
``` SQL  
Update TGT 
Set TGT.$1 = SRC.$1
From Employeestbl TGT inner join Employeestbl SRC  
	on TGT.[Email] = SRC.[Email] 
Where TGT.SkipImport = 0 And SRC.SkipImport = 1 
And (TGT.$1 is null Or TGT.$1 = 0) and SRC.$1 = 1```  

<hr/>
## Input
The user supplied the DDL for the table:
<textarea name="DDL" rows="15" cols="80">
CREATE TABLE [dbo].[Employeestbl](
	[ID] [int] IDENTITY(1,1) NOT NULL,
	[RevisionDate] [smalldatetime] NULL,
	[IntoFacility] [varchar](50) NULL,
	[EmployeeStatus] [varchar](50) NULL,
	[EmployeeStatusDate] [datetime] NULL,
	[EmployeeStatusInitial] [varchar](5) NULL,
	[LastName] [varchar](50) NULL,
	[FirstName] [varchar](50) NULL,
	[MiddleInitial] [varchar](2) NULL,
	[AddressLine1] [varchar](50) NULL,
	[AddressLine2] [varchar](50) NULL,
	[City] [varchar](30) NULL,
	[State] [char](2) NULL,
	[Zip] [varchar](12) NULL,
	[AddressLastUpdated] [datetime] NULL,
	[HomePhone] [varchar](15) NULL,
	[Workphone] [varchar](15) NULL,
	[Fax] [varchar](15) NULL,
	[Extension] [varchar](15) NULL,
	[Beeper] [varchar](20) NULL,
	[BeeperExt] [varchar](50) NULL,
	[Phone2] [varchar](15) NULL,
	[SocialSecurityNumber] [varchar](15) NULL,
	[Birthdate] [datetime] NULL,
	[Sex] [char](1) NULL,
	[Scanner] [bit] NULL,
	[MaritalStatus] [varchar](1) NULL,
	[Exemptions] [smallint] NULL,
	[HireDate] [datetime] NULL,
	[Title] [varchar](50) NOT NULL,
	[Email] [varchar](150) NULL,
	[Initial] [varchar](10) NULL,
	[WorkPermitNumber] [varchar](25) NULL,
	[WorkPermitExpires] [datetime] NULL,
	[ID_A] [varchar](50) NULL,
	[ID_A_Expires] [datetime] NULL,
	[ID_B] [varchar](50) NULL,
	[ID_B_Expires] [datetime] NULL,
	[ID_C] [varchar](50) NULL,
	[ID_C_Expires] [datetime] NULL,
	[Document_Notes] [varchar](255) NULL,
	[Document_Notes_Date] [datetime] NULL,
	[Document_Notes_Initial] [varchar](5) NULL,
	[LicenseNum] [varchar](20) NULL,
	[LicenseExpires] [datetime] NULL,
	[Physical] [datetime] NULL,
	[MalpracticeExpires] [datetime] NULL,
	[MalpracticeCompany] [varchar](50) NULL,
	[MalpracticePolicyNo] [varchar](50) NULL,
	[BclsExpires] [datetime] NULL,
	[CPR] [bit] NULL,
	[AclsExpires] [datetime] NULL,
	[NalsExpires] [datetime] NULL,
	[PalsExpires] [datetime] NULL,
	[Other_Cert] [varchar](50) NULL,
	[Other_Cert_Exp] [datetime] NULL,
	[FireSafety] [datetime] NULL,
	[InfectionControl] [datetime] NULL,
	[NoExperience] [varchar](50) NULL,
	[AddFederal] [varchar](50) NULL,
	[PhisycalPPD_Date] [datetime] NULL,
	[PhisycalPPD_Res] [varchar](50) NULL,
	[PPD2ndStepDate] [datetime] NULL,
	[PPD2ndStepRes] [varchar](50) NULL,
	[ChestXRayDate] [datetime] NULL,
	[ChestXRayRes] [varchar](50) NULL,
	[MMRImmunization] [datetime] NULL,
	[MeaslesRubeolaDate] [datetime] NULL,
	[MeaslesRubeolaRes] [varchar](50) NULL,
	[MeaslesRubeolaTiter] [bit] NULL,
	[MumpsDate] [datetime] NULL,
	[MumpsRes] [varchar](50) NULL,
	[MumpsTiter] [bit] NULL,
	[RubellaDate] [datetime] NULL,
	[RubellaRes] [varchar](50) NULL,
	[RubellaTiter] [bit] NULL,
	[VaricellaDate] [datetime] NULL,
	[VaricellaRes] [varchar](50) NULL,
	[VaricellaTiter] [bit] NULL,
	[Tetanus] [varchar](250) NULL,
	[Urinalysis] [datetime] NULL,
	[HepBVac] [datetime] NULL,
	[HepBWaiver] [datetime] NULL,
	[HepBYesNo] [bit] NULL,
	[NoNightCalls] [bit] NULL,
	[ResumeYN] [bit] NULL,
	[Application] [bit] NULL,
	[Application_Old] [bit] NULL,
	[RecOfEmploy] [bit] NULL,
	[SkillsChecklist] [bit] NULL,
	[SkillsChecklistDate] [datetime] NULL,
	[SkillsChecklistUnit1] [varchar](50) NULL,
	[SkillsChecklistUnit2] [varchar](50) NULL,
	[HospitalInterested] [bit] NULL,
	[HospitalExperience] [bit] NULL,
	[NursingHomeExperience] [bit] NULL,
	[NursingHomeInterested] [bit] NULL,
	[ReferredBy] [int] NULL,
	[ReferredByDate] [datetime] NULL,
	[TravelerYN] [bit] NULL,
	[Traveler] [varchar](50) NULL,
	[Paid] [bit] NULL,
	[PaidInitial] [varchar](5) NULL,
	[Availibility] [varchar](150) NULL,
	[AvailibilityDate] [datetime] NULL,
	[I9Complete] [bit] NULL,
	[I9Req] [bit] NULL,
	[I9OnlySig] [bit] NULL,
	[W4] [bit] NULL,
	[MapplYN] [bit] NULL,
	[Mappl] [datetime] NULL,
	[MapplInitial] [varchar](5) NULL,
	[Online] [bit] NULL,
	[Resume] [varchar](50) NULL,
	[Resume2] [varchar](150) NULL,
	[County] [varchar](25) NULL,
	[DuplicateYN] [bit] NULL,
	[TestYN] [bit] NULL,
	[Reference1] [bit] NULL,
	[Reference2] [bit] NULL,
	[Reference1Old] [bit] NULL,
	[Reference2Old] [bit] NULL,
	[BackgroundCheck] [bit] NULL,
	[DrugScreen] [datetime] NULL,
	[TovCode] [varchar](25) NULL,
	[HIPAA_Expires] [datetime] NULL,
	[Temp] [bit] NULL,
	[OP_Date] [datetime] NULL,
	[OP_Results] [varchar](15) NULL,
	[Chauncey_Date] [datetime] NULL,
	[Chauncey_Results] [varchar](15) NULL,
	[Patient_Safety_Goals] [datetime] NULL,
	[Performance_Eval_Comp] [datetime] NULL,
	[Performance_Eval_Label] [datetime] NULL,
	[Performance_Eval_Label2] [datetime] NULL,
	[Performance_Eval_Note] [varchar](150) NULL,
	[Performance_Eval_NoteDay] [datetime] NULL,
	[Performance_Eval_NoteInitial] [nvarchar](5) NULL,
	[WhiteGlove_ID] [bit] NULL,
	[Abuse] [datetime] NULL,
	[OrientationCheckList] [bit] NULL,
	[HomeAddressLine1] [varchar](75) NULL,
	[HomeAddressLine2] [varchar](75) NULL,
	[HomeCity] [varchar](150) NULL,
	[HomeState] [varchar](25) NULL,
	[HomeZip] [varchar](25) NULL,
	[EligibleToWork] [bit] NULL,
	[DayAvailToWork] [datetime] NULL,
	[FilesInitial] [varchar](5) NULL,
	[FilesDate] [datetime] NULL,
	[FilesNotes] [varchar](100) NULL,
	[FilesYesNo] [bit] NULL,
	[DocumentsYesNo] [bit] NULL,
	[EmpCode] [varchar](5) NULL,
	[EmpUserName] [varchar](75) NULL,
	[EmpPassword] [varchar](50) NULL,
	[Avail_Days] [bit] NULL,
	[Avail_Evenings] [bit] NULL,
	[Avail_Nights] [bit] NULL,
	[Avail_WeekDays] [bit] NULL,
	[Avail_Weekends] [bit] NULL,
	[Avail_Flexible] [bit] NULL,
	[Avail_8Hours] [bit] NULL,
	[Avail_12Hours] [bit] NULL,
	[Avail_Other] [bit] NULL,
	[Avail_DaysPerWeek] [int] NULL,
	[Avail_FullTime] [bit] NULL,
	[Avail_Date] [datetime] NULL,
	[Recruit_FacilityID_Initial] [varchar](5) NULL,
	[Recruit_FacilityID_Date] [datetime] NULL,
	[Recruit_FacilityID] [int] NULL,
	[Recruit_FacilityID_Initial_Entered] [varchar](50) NULL,
	[Email_Status] [varchar](50) NULL,
	[Recruitment_EmployeeID] [int] NULL,
	[FileCompleted] [bit] NULL,
	[Note] [varchar](255) NULL,
	[Print_Label] [bit] NULL,
	[Initial_Deleted] [varchar](5) NULL,
	[Reason_Deleted] [varchar](255) NULL,
	[DateEntered] [datetime] NULL,
	[HasSpecialtyLicenses_PCA] [bit] NULL,
	[HasSpecialtyLicenses_ORT] [bit] NULL,
	[HasSpecialtyLicenses_NT] [bit] NULL,
	[HasSpecialtyLicenses_HHA] [bit] NULL,
	[HasSpecialtyLicenses_PCT] [bit] NULL,
	[Streamline_Not_Received_Card] [bit] NULL,
	[SelfScheduled] [bit] NULL,
	[Bilingual] [varchar](5) NULL,
	[AvailibilityPDaysOld] [varchar](50) NULL,
	[AvailibilityPShifts] [varchar](150) NULL,
	[AvailibilityPDate] [datetime] NULL,
	[AvailibilityPDays] [varchar](50) NULL,
	[CoreMandatory] [bit] NULL,
	[CoreMandatoryDate] [datetime] NULL,
	[FileCompleteDate] [datetime] NULL,
	[FileCompleteInitial] [varchar](50) NULL,
	[References1Date] [datetime] NULL,
	[References1Initial] [varchar](50) NULL,
	[References2Date] [datetime] NULL,
	[References2Initial] [varchar](50) NULL,
	[SanctionsDate] [datetime] NULL,
	[SanctionsResults] [varchar](50) NULL,
	[OIGDate] [datetime] NULL,
	[OIGResults] [varchar](50) NULL,
	[HIPAADate] [datetime] NULL,
	[BackgroundCheckDate] [datetime] NULL,
	[EPLSDate] [datetime] NULL,
	[EPLSResults] [varchar](50) NULL,
	[ts] [timestamp] NULL,
	[HomeCare] [bit] NULL,
	[BackgroundCheckResults] [varchar](50) NULL,
	[ChauncySanctionsDate] [datetime] NULL,
	[EducationVerified] [bit] NULL,
	[WGIDStatus] [varchar](50) NULL,
	[BackgroundAgency] [varchar](50) NULL,
	[BackgroundCheckConsent] [bit] NULL,
	[ChauncySanctionsResults] [varchar](50) NULL,
	[veteran] [bit] NULL,
	[NPI] [varchar](50) NULL,
	[Performance_Eval_Comp_YN] [bit] NULL,
	[LocalContract] [bit] NULL,
	[VolSelfID] [bit] NULL,
	[SMSProvider] [varchar](50) NULL,
	[IntrestedinVAFacilities] [bit] NULL,
	[NotAvailChecked] [bit] NULL,
	[NotAvailUntil] [datetime] NULL,
	[NotSendEmail] [bit] NULL,
	[NotSendTextMsg] [bit] NULL,
	[WGDrugScreenDate] [datetime] NULL,
	[MaskFitTest] [bit] NULL,
	[FluShutDate] [datetime] NULL,
	[OrientationDocumentation] [bit] NULL,
	[ApplicationDate] [datetime] NULL,
	[H1n1] [datetime] NULL,
	[MaskFitTestDate] [datetime] NULL,
	[FluExempt] [datetime] NULL,
	[SkillsChecklistScore] [varchar](5) NULL,
	[MeaslesRubeolaLabReports] [bit] NULL,
	[MumpsLabReports] [bit] NULL,
	[RubellaLabReports] [bit] NULL,
	[VaricellaLabReports] [bit] NULL,
	[LicenseNumSignedYN] [bit] NULL,
	[FacilityCompleted] [bit] NULL,
	[FacilityCompletedDate] [datetime] NULL,
	[FacilityCompletedInitial] [varchar](5) NULL,
	[SMSStatus] [varchar](50) NULL,
	[EmailAddressInvalid] [varchar](250) NULL,
	[EmailByPhoneYN] [bit] NULL,
	[OMIGDate] [datetime] NULL,
	[OMIGResults] [varchar](50) NULL,
	[TBQDate] [datetime] NULL,
	[TBQResults] [varchar](5) NULL,
	[--NotForHomeCare--] [bit] NULL,
	[Original_Address] [varchar](150) NULL,
	[Original_City] [varchar](50) NULL,
	[Original_State] [varchar](100) NULL,
	[Original_Zip] [varchar](12) NULL,
	[ProofOfOriginalAddress] [varchar](100) NULL,
	[ProofOfOriginalAddressDate] [datetime] NULL,
	[ProofOfOriginalAddressInitial] [varchar](5) NULL,
	[CriminalPerApp] [bit] NULL,
	[JcahoColor] [varchar](50) NULL,
	[JcahoColorDate] [datetime] NULL,
	[JcahoDueDate] [datetime] NULL,
	[BSN_YN] [bit] NULL,
	[BclsSignedYN] [bit] NULL,
	[AclsSignedYN] [bit] NULL,
	[NalsSignedYN] [bit] NULL,
	[PalsSignedYN] [bit] NULL,
	[LSFormYN] [bit] NULL,
	[ResumeYNDate] [datetime] NULL,
	[FacRate] [money] NULL,
	[FacRateDate] [datetime] NULL,
	[FacRateInit] [varchar](5) NULL,
	[LSOnApplyingDate] [datetime] NULL,
	[Title2] [varchar](50) NULL,
	[SkillsChecklistScore2] [varchar](5) NULL,
	[Degree] [varchar](50) NULL,
	[SMSProviderInvalid] [varchar](50) NULL,
	[LicenseState] [varchar](5) NULL,
	[MalLevelOK] [bit] NULL,
	[NotSendMassEmail] [bit] NULL,
	[NotSendMassTextMsg] [bit] NULL,
	[AttestationFormDate] [datetime] NULL,
	[AttestationFormUploadedDate] [datetime] NULL,
	[AttestationFormSigned] [bit] NULL,
	[CorporateCompliancePolicyDate] [datetime] NULL,
	[RN_LPN_HC] [datetime] NULL,
	[SSAYN] [bit] NULL,
	[HomeCareExamsDueDate] [datetime] NULL,
	[EmailLastVerifiedDate] [datetime] NULL,
	[SMSProviderLastVerifiedDate] [datetime] NULL,
	[NSOSDate] [datetime] NULL,
	[NSOSRes] [varchar](50) NULL,
	[EmailLastVerifiedInitial] [varchar](5) NULL,
	[WGRecordsYN] [bit] NULL,
	[WGRecordsDate] [datetime] NULL,
	[WGRecordsInitial] [varchar](5) NULL,
	[VentTrainingClass] [smallint] NULL,
	[VentCertificateYN] [bit] NULL,
	[VentSupervision1YN] [bit] NULL,
	[VentSupervision2YN] [bit] NULL,
	[VentSupervision3YN] [bit] NULL,
	[FluAttestationFormDate] [datetime] NULL,
	[FluAttestationFormUploadedDate] [datetime] NULL,
	[BclsLetterDate] [datetime] NULL,
	[AclsLetterDate] [datetime] NULL,
	[NalsLetterDate] [datetime] NULL,
	[PalsLetterDate] [datetime] NULL,
	[W4Date] [datetime] NULL,
	[EthnicGroup] [varchar](50) NULL,
	[SizeModel] [varchar](50) NULL,
	[ExpectedGraduation] [varchar](50) NULL,
	[ExpectedGraduationDate] [datetime] NULL,
	[EVerifyYN] [bit] NULL,
	[EmailVerifiedByVendorDate] [datetime] NULL,
	[MedicalClearanceYN] [bit] NULL,
	[MedicalClearanceFacility] [varchar](150) NULL,
	[InsuranceStatus] [varchar](50) NULL,
	[BestTimeToReach] [varchar](50) NULL,
	[ReferredByName] [varchar](100) NULL,
	[TaxCredit8850YN] [bit] NULL,
	[TaxCredit8850] [varchar](50) NULL,
	[ResumeYNInitial] [varchar](5) NULL,
	[COIDate] [datetime] NULL,
	[OrientationDocumentationFacility] [varchar](150) NULL,
	[CPIDate] [datetime] NULL,
	[HepBTiter] [bit] NULL,
	[HepBLabReports] [bit] NULL,
	[HepBRes] [varchar](50) NULL,
	[OmigOigSamDate] [datetime] NULL,
	[OmigOigSamRes] [varchar](50) NULL,
	[DriveYN] [int] NULL,
	[Email2] [varchar](150) NULL,
	[Email2VerifiedByVendorDate] [datetime] NULL,
	[Email2_Status] [varchar](50) NULL,
	[ORTVerificationDate] [datetime] NULL,
	[ORTResults] [varchar](50) NULL,
	[HepBDeclYN] [bit] NULL,
	[TBQuantYN] [bit] NULL,
	[EverifiedDate] [datetime] NULL,
	[EducationVerifiedDate] [datetime] NULL,
	[EducationVerifiedType] [varchar](50) NULL,
	[SSNSearch] [bit] NULL,
	[Fingerprint] [varchar](50) NULL,
	[DateExportedBS] [datetime] NULL,
	[SkipImport] [int] NULL,
 CONSTRAINT [PK_Employeestbl] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX  = OFF, STATISTICS_NORECOMPUTE  = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS  = ON, ALLOW_PAGE_LOCKS  = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
</textarea>  

<hr>
## Process
Here's where regular expressions can really help you do some templating magic.  In Notepad++, you need to set the _Search Mode_ to "Regular Expression" when you display the Find/Replace dialog.
1. I have to delete the non-field-defining lines.  
<textarea name="DDLprep1" rows="10" cols="80">
	CREATE TABLE [dbo].[Employeestbl](
	 CONSTRAINT [PK_Employeestbl] PRIMARY KEY CLUSTERED 
	(
		[ID] ASC
	)WITH (PAD_INDEX  = OFF, STATISTICS_NORECOMPUTE  = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS  = ON, ALLOW_PAGE_LOCKS  = ON) ON [PRIMARY]
	) ON [PRIMARY]

	GO
</textarea>  
I initially did this manually, but the Find/Replace commands are:
Find what: __CONSTRAINT[^$]+__  
Replace with:

Find what: __CREATE TABLE [^\r]+\r\n__  
Replace with:

2. I have to delete the ID and SkipImport lines.
<textarea name="DDLprep2" rows="3" cols="80">
	[ID] [int] IDENTITY(1,1) NOT NULL,
	[SkipImport] [int] NULL,
</textarea>  
I initially did this manually, but the Find/Replace commands are:  
Find what: __\t\[(ID|SkipImport)\][^\r]+\r\n__  
Replace with:

The remaining DDL lines are transformed with one Find/Replace operation per type of column (date, bit, everything else).  As with such processing, exceptions (special cases) are processed first.  The Update statements (above) are modified so that they are on a single line.

Find what: __\t(\[.+?\]) \[(?:smalldatetime|datetime)\] [^,]+,__  
Replace with: __Update TGT Set TGT.$1 = SRC.$1 From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And \(\(TGT.$1 is null And SRC.$1 is not null\) OR \(TGT.$1 is not null And SRC.$1 is not null And TGT.$1 < SRC.$1\)\);__  

Find what: __\t(\[.+?\]) \[bit\] [^,]+,__  
Replace with: __Update TGT Set TGT.$1 = SRC.$1 From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And \(TGT.$1 is null Or TGT.$1 = 0\) and SRC.$1 = 1;__  

Find what: __\t(\[.+?\]) \[[^,]+,__  
Replace with: __Update TGT Set TGT.$1 = SRC.$1 From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.$1 is null and SRC.$1 is not null;__  

__Notes:__  
* Removing the leading tab character with the edit gives good visual clue for
checking the find/replace results
	
* Since the result is > 64K	characters, the resulting SQL has to be
broken up into two executions.

<hr>
<textarea name="transformedDDL" rows="15" cols="80">
Update TGT Set TGT.[RevisionDate] = SRC.[RevisionDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[RevisionDate] is null And SRC.[RevisionDate] is not null) OR (TGT.[RevisionDate] is not null And SRC.[RevisionDate] is not null And TGT.[RevisionDate] < SRC.[RevisionDate]));
Update TGT Set TGT.[IntoFacility] = SRC.[IntoFacility] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[IntoFacility] is null and SRC.[IntoFacility] is not null;
Update TGT Set TGT.[EmployeeStatus] = SRC.[EmployeeStatus] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[EmployeeStatus] is null and SRC.[EmployeeStatus] is not null;
Update TGT Set TGT.[EmployeeStatusDate] = SRC.[EmployeeStatusDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[EmployeeStatusDate] is null And SRC.[EmployeeStatusDate] is not null) OR (TGT.[EmployeeStatusDate] is not null And SRC.[EmployeeStatusDate] is not null And TGT.[EmployeeStatusDate] < SRC.[EmployeeStatusDate]));
Update TGT Set TGT.[EmployeeStatusInitial] = SRC.[EmployeeStatusInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[EmployeeStatusInitial] is null and SRC.[EmployeeStatusInitial] is not null;
Update TGT Set TGT.[LastName] = SRC.[LastName] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[LastName] is null and SRC.[LastName] is not null;
Update TGT Set TGT.[FirstName] = SRC.[FirstName] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[FirstName] is null and SRC.[FirstName] is not null;
Update TGT Set TGT.[MiddleInitial] = SRC.[MiddleInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[MiddleInitial] is null and SRC.[MiddleInitial] is not null;
Update TGT Set TGT.[AddressLine1] = SRC.[AddressLine1] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[AddressLine1] is null and SRC.[AddressLine1] is not null;
Update TGT Set TGT.[AddressLine2] = SRC.[AddressLine2] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[AddressLine2] is null and SRC.[AddressLine2] is not null;
Update TGT Set TGT.[City] = SRC.[City] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[City] is null and SRC.[City] is not null;
Update TGT Set TGT.[State] = SRC.[State] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[State] is null and SRC.[State] is not null;
Update TGT Set TGT.[Zip] = SRC.[Zip] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Zip] is null and SRC.[Zip] is not null;
Update TGT Set TGT.[AddressLastUpdated] = SRC.[AddressLastUpdated] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[AddressLastUpdated] is null And SRC.[AddressLastUpdated] is not null) OR (TGT.[AddressLastUpdated] is not null And SRC.[AddressLastUpdated] is not null And TGT.[AddressLastUpdated] < SRC.[AddressLastUpdated]));
Update TGT Set TGT.[HomePhone] = SRC.[HomePhone] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[HomePhone] is null and SRC.[HomePhone] is not null;
Update TGT Set TGT.[Workphone] = SRC.[Workphone] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Workphone] is null and SRC.[Workphone] is not null;
Update TGT Set TGT.[Fax] = SRC.[Fax] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Fax] is null and SRC.[Fax] is not null;
Update TGT Set TGT.[Extension] = SRC.[Extension] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Extension] is null and SRC.[Extension] is not null;
Update TGT Set TGT.[Beeper] = SRC.[Beeper] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Beeper] is null and SRC.[Beeper] is not null;
Update TGT Set TGT.[BeeperExt] = SRC.[BeeperExt] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[BeeperExt] is null and SRC.[BeeperExt] is not null;
Update TGT Set TGT.[Phone2] = SRC.[Phone2] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Phone2] is null and SRC.[Phone2] is not null;
Update TGT Set TGT.[SocialSecurityNumber] = SRC.[SocialSecurityNumber] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[SocialSecurityNumber] is null and SRC.[SocialSecurityNumber] is not null;
Update TGT Set TGT.[Birthdate] = SRC.[Birthdate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Birthdate] is null And SRC.[Birthdate] is not null) OR (TGT.[Birthdate] is not null And SRC.[Birthdate] is not null And TGT.[Birthdate] < SRC.[Birthdate]));
Update TGT Set TGT.[Sex] = SRC.[Sex] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Sex] is null and SRC.[Sex] is not null;
Update TGT Set TGT.[Scanner] = SRC.[Scanner] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Scanner] is null Or TGT.[Scanner] = 0) and SRC.[Scanner] = 1;
Update TGT Set TGT.[MaritalStatus] = SRC.[MaritalStatus] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[MaritalStatus] is null and SRC.[MaritalStatus] is not null;
Update TGT Set TGT.[Exemptions] = SRC.[Exemptions] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Exemptions] is null and SRC.[Exemptions] is not null;
Update TGT Set TGT.[HireDate] = SRC.[HireDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[HireDate] is null And SRC.[HireDate] is not null) OR (TGT.[HireDate] is not null And SRC.[HireDate] is not null And TGT.[HireDate] < SRC.[HireDate]));
Update TGT Set TGT.[Title] = SRC.[Title] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Title] is null and SRC.[Title] is not null;
Update TGT Set TGT.[Email] = SRC.[Email] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Email] is null and SRC.[Email] is not null;
Update TGT Set TGT.[Initial] = SRC.[Initial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Initial] is null and SRC.[Initial] is not null;
Update TGT Set TGT.[WorkPermitNumber] = SRC.[WorkPermitNumber] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[WorkPermitNumber] is null and SRC.[WorkPermitNumber] is not null;
Update TGT Set TGT.[WorkPermitExpires] = SRC.[WorkPermitExpires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[WorkPermitExpires] is null And SRC.[WorkPermitExpires] is not null) OR (TGT.[WorkPermitExpires] is not null And SRC.[WorkPermitExpires] is not null And TGT.[WorkPermitExpires] < SRC.[WorkPermitExpires]));
Update TGT Set TGT.[ID_A] = SRC.[ID_A] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ID_A] is null and SRC.[ID_A] is not null;
Update TGT Set TGT.[ID_A_Expires] = SRC.[ID_A_Expires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ID_A_Expires] is null And SRC.[ID_A_Expires] is not null) OR (TGT.[ID_A_Expires] is not null And SRC.[ID_A_Expires] is not null And TGT.[ID_A_Expires] < SRC.[ID_A_Expires]));
Update TGT Set TGT.[ID_B] = SRC.[ID_B] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ID_B] is null and SRC.[ID_B] is not null;
Update TGT Set TGT.[ID_B_Expires] = SRC.[ID_B_Expires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ID_B_Expires] is null And SRC.[ID_B_Expires] is not null) OR (TGT.[ID_B_Expires] is not null And SRC.[ID_B_Expires] is not null And TGT.[ID_B_Expires] < SRC.[ID_B_Expires]));
Update TGT Set TGT.[ID_C] = SRC.[ID_C] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ID_C] is null and SRC.[ID_C] is not null;
Update TGT Set TGT.[ID_C_Expires] = SRC.[ID_C_Expires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ID_C_Expires] is null And SRC.[ID_C_Expires] is not null) OR (TGT.[ID_C_Expires] is not null And SRC.[ID_C_Expires] is not null And TGT.[ID_C_Expires] < SRC.[ID_C_Expires]));
Update TGT Set TGT.[Document_Notes] = SRC.[Document_Notes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Document_Notes] is null and SRC.[Document_Notes] is not null;
Update TGT Set TGT.[Document_Notes_Date] = SRC.[Document_Notes_Date] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Document_Notes_Date] is null And SRC.[Document_Notes_Date] is not null) OR (TGT.[Document_Notes_Date] is not null And SRC.[Document_Notes_Date] is not null And TGT.[Document_Notes_Date] < SRC.[Document_Notes_Date]));
Update TGT Set TGT.[Document_Notes_Initial] = SRC.[Document_Notes_Initial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Document_Notes_Initial] is null and SRC.[Document_Notes_Initial] is not null;
Update TGT Set TGT.[LicenseNum] = SRC.[LicenseNum] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[LicenseNum] is null and SRC.[LicenseNum] is not null;
Update TGT Set TGT.[LicenseExpires] = SRC.[LicenseExpires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[LicenseExpires] is null And SRC.[LicenseExpires] is not null) OR (TGT.[LicenseExpires] is not null And SRC.[LicenseExpires] is not null And TGT.[LicenseExpires] < SRC.[LicenseExpires]));
Update TGT Set TGT.[Physical] = SRC.[Physical] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Physical] is null And SRC.[Physical] is not null) OR (TGT.[Physical] is not null And SRC.[Physical] is not null And TGT.[Physical] < SRC.[Physical]));
Update TGT Set TGT.[MalpracticeExpires] = SRC.[MalpracticeExpires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[MalpracticeExpires] is null And SRC.[MalpracticeExpires] is not null) OR (TGT.[MalpracticeExpires] is not null And SRC.[MalpracticeExpires] is not null And TGT.[MalpracticeExpires] < SRC.[MalpracticeExpires]));
Update TGT Set TGT.[MalpracticeCompany] = SRC.[MalpracticeCompany] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[MalpracticeCompany] is null and SRC.[MalpracticeCompany] is not null;
Update TGT Set TGT.[MalpracticePolicyNo] = SRC.[MalpracticePolicyNo] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[MalpracticePolicyNo] is null and SRC.[MalpracticePolicyNo] is not null;
Update TGT Set TGT.[BclsExpires] = SRC.[BclsExpires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[BclsExpires] is null And SRC.[BclsExpires] is not null) OR (TGT.[BclsExpires] is not null And SRC.[BclsExpires] is not null And TGT.[BclsExpires] < SRC.[BclsExpires]));
Update TGT Set TGT.[CPR] = SRC.[CPR] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[CPR] is null Or TGT.[CPR] = 0) and SRC.[CPR] = 1;
Update TGT Set TGT.[AclsExpires] = SRC.[AclsExpires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[AclsExpires] is null And SRC.[AclsExpires] is not null) OR (TGT.[AclsExpires] is not null And SRC.[AclsExpires] is not null And TGT.[AclsExpires] < SRC.[AclsExpires]));
Update TGT Set TGT.[NalsExpires] = SRC.[NalsExpires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[NalsExpires] is null And SRC.[NalsExpires] is not null) OR (TGT.[NalsExpires] is not null And SRC.[NalsExpires] is not null And TGT.[NalsExpires] < SRC.[NalsExpires]));
Update TGT Set TGT.[PalsExpires] = SRC.[PalsExpires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[PalsExpires] is null And SRC.[PalsExpires] is not null) OR (TGT.[PalsExpires] is not null And SRC.[PalsExpires] is not null And TGT.[PalsExpires] < SRC.[PalsExpires]));
Update TGT Set TGT.[Other_Cert] = SRC.[Other_Cert] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Other_Cert] is null and SRC.[Other_Cert] is not null;
Update TGT Set TGT.[Other_Cert_Exp] = SRC.[Other_Cert_Exp] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Other_Cert_Exp] is null And SRC.[Other_Cert_Exp] is not null) OR (TGT.[Other_Cert_Exp] is not null And SRC.[Other_Cert_Exp] is not null And TGT.[Other_Cert_Exp] < SRC.[Other_Cert_Exp]));
Update TGT Set TGT.[FireSafety] = SRC.[FireSafety] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[FireSafety] is null And SRC.[FireSafety] is not null) OR (TGT.[FireSafety] is not null And SRC.[FireSafety] is not null And TGT.[FireSafety] < SRC.[FireSafety]));
Update TGT Set TGT.[InfectionControl] = SRC.[InfectionControl] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[InfectionControl] is null And SRC.[InfectionControl] is not null) OR (TGT.[InfectionControl] is not null And SRC.[InfectionControl] is not null And TGT.[InfectionControl] < SRC.[InfectionControl]));
Update TGT Set TGT.[NoExperience] = SRC.[NoExperience] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[NoExperience] is null and SRC.[NoExperience] is not null;
Update TGT Set TGT.[AddFederal] = SRC.[AddFederal] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[AddFederal] is null and SRC.[AddFederal] is not null;
Update TGT Set TGT.[PhisycalPPD_Date] = SRC.[PhisycalPPD_Date] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[PhisycalPPD_Date] is null And SRC.[PhisycalPPD_Date] is not null) OR (TGT.[PhisycalPPD_Date] is not null And SRC.[PhisycalPPD_Date] is not null And TGT.[PhisycalPPD_Date] < SRC.[PhisycalPPD_Date]));
Update TGT Set TGT.[PhisycalPPD_Res] = SRC.[PhisycalPPD_Res] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[PhisycalPPD_Res] is null and SRC.[PhisycalPPD_Res] is not null;
Update TGT Set TGT.[PPD2ndStepDate] = SRC.[PPD2ndStepDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[PPD2ndStepDate] is null And SRC.[PPD2ndStepDate] is not null) OR (TGT.[PPD2ndStepDate] is not null And SRC.[PPD2ndStepDate] is not null And TGT.[PPD2ndStepDate] < SRC.[PPD2ndStepDate]));
Update TGT Set TGT.[PPD2ndStepRes] = SRC.[PPD2ndStepRes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[PPD2ndStepRes] is null and SRC.[PPD2ndStepRes] is not null;
Update TGT Set TGT.[ChestXRayDate] = SRC.[ChestXRayDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ChestXRayDate] is null And SRC.[ChestXRayDate] is not null) OR (TGT.[ChestXRayDate] is not null And SRC.[ChestXRayDate] is not null And TGT.[ChestXRayDate] < SRC.[ChestXRayDate]));
Update TGT Set TGT.[ChestXRayRes] = SRC.[ChestXRayRes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ChestXRayRes] is null and SRC.[ChestXRayRes] is not null;
Update TGT Set TGT.[MMRImmunization] = SRC.[MMRImmunization] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[MMRImmunization] is null And SRC.[MMRImmunization] is not null) OR (TGT.[MMRImmunization] is not null And SRC.[MMRImmunization] is not null And TGT.[MMRImmunization] < SRC.[MMRImmunization]));
Update TGT Set TGT.[MeaslesRubeolaDate] = SRC.[MeaslesRubeolaDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[MeaslesRubeolaDate] is null And SRC.[MeaslesRubeolaDate] is not null) OR (TGT.[MeaslesRubeolaDate] is not null And SRC.[MeaslesRubeolaDate] is not null And TGT.[MeaslesRubeolaDate] < SRC.[MeaslesRubeolaDate]));
Update TGT Set TGT.[MeaslesRubeolaRes] = SRC.[MeaslesRubeolaRes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[MeaslesRubeolaRes] is null and SRC.[MeaslesRubeolaRes] is not null;
Update TGT Set TGT.[MeaslesRubeolaTiter] = SRC.[MeaslesRubeolaTiter] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[MeaslesRubeolaTiter] is null Or TGT.[MeaslesRubeolaTiter] = 0) and SRC.[MeaslesRubeolaTiter] = 1;
Update TGT Set TGT.[MumpsDate] = SRC.[MumpsDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[MumpsDate] is null And SRC.[MumpsDate] is not null) OR (TGT.[MumpsDate] is not null And SRC.[MumpsDate] is not null And TGT.[MumpsDate] < SRC.[MumpsDate]));
Update TGT Set TGT.[MumpsRes] = SRC.[MumpsRes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[MumpsRes] is null and SRC.[MumpsRes] is not null;
Update TGT Set TGT.[MumpsTiter] = SRC.[MumpsTiter] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[MumpsTiter] is null Or TGT.[MumpsTiter] = 0) and SRC.[MumpsTiter] = 1;
Update TGT Set TGT.[RubellaDate] = SRC.[RubellaDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[RubellaDate] is null And SRC.[RubellaDate] is not null) OR (TGT.[RubellaDate] is not null And SRC.[RubellaDate] is not null And TGT.[RubellaDate] < SRC.[RubellaDate]));
Update TGT Set TGT.[RubellaRes] = SRC.[RubellaRes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[RubellaRes] is null and SRC.[RubellaRes] is not null;
Update TGT Set TGT.[RubellaTiter] = SRC.[RubellaTiter] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[RubellaTiter] is null Or TGT.[RubellaTiter] = 0) and SRC.[RubellaTiter] = 1;
Update TGT Set TGT.[VaricellaDate] = SRC.[VaricellaDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[VaricellaDate] is null And SRC.[VaricellaDate] is not null) OR (TGT.[VaricellaDate] is not null And SRC.[VaricellaDate] is not null And TGT.[VaricellaDate] < SRC.[VaricellaDate]));
Update TGT Set TGT.[VaricellaRes] = SRC.[VaricellaRes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[VaricellaRes] is null and SRC.[VaricellaRes] is not null;
Update TGT Set TGT.[VaricellaTiter] = SRC.[VaricellaTiter] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[VaricellaTiter] is null Or TGT.[VaricellaTiter] = 0) and SRC.[VaricellaTiter] = 1;
Update TGT Set TGT.[Tetanus] = SRC.[Tetanus] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Tetanus] is null and SRC.[Tetanus] is not null;
Update TGT Set TGT.[Urinalysis] = SRC.[Urinalysis] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Urinalysis] is null And SRC.[Urinalysis] is not null) OR (TGT.[Urinalysis] is not null And SRC.[Urinalysis] is not null And TGT.[Urinalysis] < SRC.[Urinalysis]));
Update TGT Set TGT.[HepBVac] = SRC.[HepBVac] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[HepBVac] is null And SRC.[HepBVac] is not null) OR (TGT.[HepBVac] is not null And SRC.[HepBVac] is not null And TGT.[HepBVac] < SRC.[HepBVac]));
Update TGT Set TGT.[HepBWaiver] = SRC.[HepBWaiver] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[HepBWaiver] is null And SRC.[HepBWaiver] is not null) OR (TGT.[HepBWaiver] is not null And SRC.[HepBWaiver] is not null And TGT.[HepBWaiver] < SRC.[HepBWaiver]));
Update TGT Set TGT.[HepBYesNo] = SRC.[HepBYesNo] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HepBYesNo] is null Or TGT.[HepBYesNo] = 0) and SRC.[HepBYesNo] = 1;
Update TGT Set TGT.[NoNightCalls] = SRC.[NoNightCalls] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[NoNightCalls] is null Or TGT.[NoNightCalls] = 0) and SRC.[NoNightCalls] = 1;
Update TGT Set TGT.[ResumeYN] = SRC.[ResumeYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[ResumeYN] is null Or TGT.[ResumeYN] = 0) and SRC.[ResumeYN] = 1;
Update TGT Set TGT.[Application] = SRC.[Application] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Application] is null Or TGT.[Application] = 0) and SRC.[Application] = 1;
Update TGT Set TGT.[Application_Old] = SRC.[Application_Old] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Application_Old] is null Or TGT.[Application_Old] = 0) and SRC.[Application_Old] = 1;
Update TGT Set TGT.[RecOfEmploy] = SRC.[RecOfEmploy] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[RecOfEmploy] is null Or TGT.[RecOfEmploy] = 0) and SRC.[RecOfEmploy] = 1;
Update TGT Set TGT.[SkillsChecklist] = SRC.[SkillsChecklist] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[SkillsChecklist] is null Or TGT.[SkillsChecklist] = 0) and SRC.[SkillsChecklist] = 1;
Update TGT Set TGT.[SkillsChecklistDate] = SRC.[SkillsChecklistDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[SkillsChecklistDate] is null And SRC.[SkillsChecklistDate] is not null) OR (TGT.[SkillsChecklistDate] is not null And SRC.[SkillsChecklistDate] is not null And TGT.[SkillsChecklistDate] < SRC.[SkillsChecklistDate]));
Update TGT Set TGT.[SkillsChecklistUnit1] = SRC.[SkillsChecklistUnit1] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[SkillsChecklistUnit1] is null and SRC.[SkillsChecklistUnit1] is not null;
Update TGT Set TGT.[SkillsChecklistUnit2] = SRC.[SkillsChecklistUnit2] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[SkillsChecklistUnit2] is null and SRC.[SkillsChecklistUnit2] is not null;
Update TGT Set TGT.[HospitalInterested] = SRC.[HospitalInterested] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HospitalInterested] is null Or TGT.[HospitalInterested] = 0) and SRC.[HospitalInterested] = 1;
Update TGT Set TGT.[HospitalExperience] = SRC.[HospitalExperience] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HospitalExperience] is null Or TGT.[HospitalExperience] = 0) and SRC.[HospitalExperience] = 1;
Update TGT Set TGT.[NursingHomeExperience] = SRC.[NursingHomeExperience] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[NursingHomeExperience] is null Or TGT.[NursingHomeExperience] = 0) and SRC.[NursingHomeExperience] = 1;
Update TGT Set TGT.[NursingHomeInterested] = SRC.[NursingHomeInterested] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[NursingHomeInterested] is null Or TGT.[NursingHomeInterested] = 0) and SRC.[NursingHomeInterested] = 1;
Update TGT Set TGT.[ReferredBy] = SRC.[ReferredBy] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ReferredBy] is null and SRC.[ReferredBy] is not null;
Update TGT Set TGT.[ReferredByDate] = SRC.[ReferredByDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ReferredByDate] is null And SRC.[ReferredByDate] is not null) OR (TGT.[ReferredByDate] is not null And SRC.[ReferredByDate] is not null And TGT.[ReferredByDate] < SRC.[ReferredByDate]));
Update TGT Set TGT.[TravelerYN] = SRC.[TravelerYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[TravelerYN] is null Or TGT.[TravelerYN] = 0) and SRC.[TravelerYN] = 1;
Update TGT Set TGT.[Traveler] = SRC.[Traveler] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Traveler] is null and SRC.[Traveler] is not null;
Update TGT Set TGT.[Paid] = SRC.[Paid] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Paid] is null Or TGT.[Paid] = 0) and SRC.[Paid] = 1;
Update TGT Set TGT.[PaidInitial] = SRC.[PaidInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[PaidInitial] is null and SRC.[PaidInitial] is not null;
Update TGT Set TGT.[Availibility] = SRC.[Availibility] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Availibility] is null and SRC.[Availibility] is not null;
Update TGT Set TGT.[AvailibilityDate] = SRC.[AvailibilityDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[AvailibilityDate] is null And SRC.[AvailibilityDate] is not null) OR (TGT.[AvailibilityDate] is not null And SRC.[AvailibilityDate] is not null And TGT.[AvailibilityDate] < SRC.[AvailibilityDate]));
Update TGT Set TGT.[I9Complete] = SRC.[I9Complete] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[I9Complete] is null Or TGT.[I9Complete] = 0) and SRC.[I9Complete] = 1;
Update TGT Set TGT.[I9Req] = SRC.[I9Req] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[I9Req] is null Or TGT.[I9Req] = 0) and SRC.[I9Req] = 1;
Update TGT Set TGT.[I9OnlySig] = SRC.[I9OnlySig] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[I9OnlySig] is null Or TGT.[I9OnlySig] = 0) and SRC.[I9OnlySig] = 1;
Update TGT Set TGT.[W4] = SRC.[W4] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[W4] is null Or TGT.[W4] = 0) and SRC.[W4] = 1;
Update TGT Set TGT.[MapplYN] = SRC.[MapplYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[MapplYN] is null Or TGT.[MapplYN] = 0) and SRC.[MapplYN] = 1;
Update TGT Set TGT.[Mappl] = SRC.[Mappl] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Mappl] is null And SRC.[Mappl] is not null) OR (TGT.[Mappl] is not null And SRC.[Mappl] is not null And TGT.[Mappl] < SRC.[Mappl]));
Update TGT Set TGT.[MapplInitial] = SRC.[MapplInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[MapplInitial] is null and SRC.[MapplInitial] is not null;
Update TGT Set TGT.[Online] = SRC.[Online] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Online] is null Or TGT.[Online] = 0) and SRC.[Online] = 1;
Update TGT Set TGT.[Resume] = SRC.[Resume] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Resume] is null and SRC.[Resume] is not null;
Update TGT Set TGT.[Resume2] = SRC.[Resume2] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Resume2] is null and SRC.[Resume2] is not null;
Update TGT Set TGT.[County] = SRC.[County] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[County] is null and SRC.[County] is not null;
Update TGT Set TGT.[DuplicateYN] = SRC.[DuplicateYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[DuplicateYN] is null Or TGT.[DuplicateYN] = 0) and SRC.[DuplicateYN] = 1;
Update TGT Set TGT.[TestYN] = SRC.[TestYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[TestYN] is null Or TGT.[TestYN] = 0) and SRC.[TestYN] = 1;
Update TGT Set TGT.[Reference1] = SRC.[Reference1] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Reference1] is null Or TGT.[Reference1] = 0) and SRC.[Reference1] = 1;
Update TGT Set TGT.[Reference2] = SRC.[Reference2] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Reference2] is null Or TGT.[Reference2] = 0) and SRC.[Reference2] = 1;
Update TGT Set TGT.[Reference1Old] = SRC.[Reference1Old] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Reference1Old] is null Or TGT.[Reference1Old] = 0) and SRC.[Reference1Old] = 1;
Update TGT Set TGT.[Reference2Old] = SRC.[Reference2Old] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Reference2Old] is null Or TGT.[Reference2Old] = 0) and SRC.[Reference2Old] = 1;
Update TGT Set TGT.[BackgroundCheck] = SRC.[BackgroundCheck] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[BackgroundCheck] is null Or TGT.[BackgroundCheck] = 0) and SRC.[BackgroundCheck] = 1;
Update TGT Set TGT.[DrugScreen] = SRC.[DrugScreen] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[DrugScreen] is null And SRC.[DrugScreen] is not null) OR (TGT.[DrugScreen] is not null And SRC.[DrugScreen] is not null And TGT.[DrugScreen] < SRC.[DrugScreen]));
Update TGT Set TGT.[TovCode] = SRC.[TovCode] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[TovCode] is null and SRC.[TovCode] is not null;
Update TGT Set TGT.[HIPAA_Expires] = SRC.[HIPAA_Expires] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[HIPAA_Expires] is null And SRC.[HIPAA_Expires] is not null) OR (TGT.[HIPAA_Expires] is not null And SRC.[HIPAA_Expires] is not null And TGT.[HIPAA_Expires] < SRC.[HIPAA_Expires]));
Update TGT Set TGT.[Temp] = SRC.[Temp] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Temp] is null Or TGT.[Temp] = 0) and SRC.[Temp] = 1;
Update TGT Set TGT.[OP_Date] = SRC.[OP_Date] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[OP_Date] is null And SRC.[OP_Date] is not null) OR (TGT.[OP_Date] is not null And SRC.[OP_Date] is not null And TGT.[OP_Date] < SRC.[OP_Date]));
Update TGT Set TGT.[OP_Results] = SRC.[OP_Results] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[OP_Results] is null and SRC.[OP_Results] is not null;
Update TGT Set TGT.[Chauncey_Date] = SRC.[Chauncey_Date] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Chauncey_Date] is null And SRC.[Chauncey_Date] is not null) OR (TGT.[Chauncey_Date] is not null And SRC.[Chauncey_Date] is not null And TGT.[Chauncey_Date] < SRC.[Chauncey_Date]));
Update TGT Set TGT.[Chauncey_Results] = SRC.[Chauncey_Results] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Chauncey_Results] is null and SRC.[Chauncey_Results] is not null;
Update TGT Set TGT.[Patient_Safety_Goals] = SRC.[Patient_Safety_Goals] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Patient_Safety_Goals] is null And SRC.[Patient_Safety_Goals] is not null) OR (TGT.[Patient_Safety_Goals] is not null And SRC.[Patient_Safety_Goals] is not null And TGT.[Patient_Safety_Goals] < SRC.[Patient_Safety_Goals]));
Update TGT Set TGT.[Performance_Eval_Comp] = SRC.[Performance_Eval_Comp] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Performance_Eval_Comp] is null And SRC.[Performance_Eval_Comp] is not null) OR (TGT.[Performance_Eval_Comp] is not null And SRC.[Performance_Eval_Comp] is not null And TGT.[Performance_Eval_Comp] < SRC.[Performance_Eval_Comp]));
Update TGT Set TGT.[Performance_Eval_Label] = SRC.[Performance_Eval_Label] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Performance_Eval_Label] is null And SRC.[Performance_Eval_Label] is not null) OR (TGT.[Performance_Eval_Label] is not null And SRC.[Performance_Eval_Label] is not null And TGT.[Performance_Eval_Label] < SRC.[Performance_Eval_Label]));
Update TGT Set TGT.[Performance_Eval_Label2] = SRC.[Performance_Eval_Label2] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Performance_Eval_Label2] is null And SRC.[Performance_Eval_Label2] is not null) OR (TGT.[Performance_Eval_Label2] is not null And SRC.[Performance_Eval_Label2] is not null And TGT.[Performance_Eval_Label2] < SRC.[Performance_Eval_Label2]));
Update TGT Set TGT.[Performance_Eval_Note] = SRC.[Performance_Eval_Note] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Performance_Eval_Note] is null and SRC.[Performance_Eval_Note] is not null;
Update TGT Set TGT.[Performance_Eval_NoteDay] = SRC.[Performance_Eval_NoteDay] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Performance_Eval_NoteDay] is null And SRC.[Performance_Eval_NoteDay] is not null) OR (TGT.[Performance_Eval_NoteDay] is not null And SRC.[Performance_Eval_NoteDay] is not null And TGT.[Performance_Eval_NoteDay] < SRC.[Performance_Eval_NoteDay]));
Update TGT Set TGT.[Performance_Eval_NoteInitial] = SRC.[Performance_Eval_NoteInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Performance_Eval_NoteInitial] is null and SRC.[Performance_Eval_NoteInitial] is not null;
Update TGT Set TGT.[WhiteGlove_ID] = SRC.[WhiteGlove_ID] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[WhiteGlove_ID] is null Or TGT.[WhiteGlove_ID] = 0) and SRC.[WhiteGlove_ID] = 1;
Update TGT Set TGT.[Abuse] = SRC.[Abuse] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Abuse] is null And SRC.[Abuse] is not null) OR (TGT.[Abuse] is not null And SRC.[Abuse] is not null And TGT.[Abuse] < SRC.[Abuse]));
Update TGT Set TGT.[OrientationCheckList] = SRC.[OrientationCheckList] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[OrientationCheckList] is null Or TGT.[OrientationCheckList] = 0) and SRC.[OrientationCheckList] = 1;
Update TGT Set TGT.[HomeAddressLine1] = SRC.[HomeAddressLine1] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[HomeAddressLine1] is null and SRC.[HomeAddressLine1] is not null;
Update TGT Set TGT.[HomeAddressLine2] = SRC.[HomeAddressLine2] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[HomeAddressLine2] is null and SRC.[HomeAddressLine2] is not null;
Update TGT Set TGT.[HomeCity] = SRC.[HomeCity] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[HomeCity] is null and SRC.[HomeCity] is not null;
Update TGT Set TGT.[HomeState] = SRC.[HomeState] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[HomeState] is null and SRC.[HomeState] is not null;
Update TGT Set TGT.[HomeZip] = SRC.[HomeZip] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[HomeZip] is null and SRC.[HomeZip] is not null;
Update TGT Set TGT.[EligibleToWork] = SRC.[EligibleToWork] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[EligibleToWork] is null Or TGT.[EligibleToWork] = 0) and SRC.[EligibleToWork] = 1;
Update TGT Set TGT.[DayAvailToWork] = SRC.[DayAvailToWork] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[DayAvailToWork] is null And SRC.[DayAvailToWork] is not null) OR (TGT.[DayAvailToWork] is not null And SRC.[DayAvailToWork] is not null And TGT.[DayAvailToWork] < SRC.[DayAvailToWork]));
Update TGT Set TGT.[FilesInitial] = SRC.[FilesInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[FilesInitial] is null and SRC.[FilesInitial] is not null;
Update TGT Set TGT.[FilesDate] = SRC.[FilesDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[FilesDate] is null And SRC.[FilesDate] is not null) OR (TGT.[FilesDate] is not null And SRC.[FilesDate] is not null And TGT.[FilesDate] < SRC.[FilesDate]));
Update TGT Set TGT.[FilesNotes] = SRC.[FilesNotes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[FilesNotes] is null and SRC.[FilesNotes] is not null;
Update TGT Set TGT.[FilesYesNo] = SRC.[FilesYesNo] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[FilesYesNo] is null Or TGT.[FilesYesNo] = 0) and SRC.[FilesYesNo] = 1;
Update TGT Set TGT.[DocumentsYesNo] = SRC.[DocumentsYesNo] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[DocumentsYesNo] is null Or TGT.[DocumentsYesNo] = 0) and SRC.[DocumentsYesNo] = 1;
Update TGT Set TGT.[EmpCode] = SRC.[EmpCode] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[EmpCode] is null and SRC.[EmpCode] is not null;
Update TGT Set TGT.[EmpUserName] = SRC.[EmpUserName] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[EmpUserName] is null and SRC.[EmpUserName] is not null;
Update TGT Set TGT.[EmpPassword] = SRC.[EmpPassword] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[EmpPassword] is null and SRC.[EmpPassword] is not null;
Update TGT Set TGT.[Avail_Days] = SRC.[Avail_Days] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Avail_Days] is null Or TGT.[Avail_Days] = 0) and SRC.[Avail_Days] = 1;
Update TGT Set TGT.[Avail_Evenings] = SRC.[Avail_Evenings] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Avail_Evenings] is null Or TGT.[Avail_Evenings] = 0) and SRC.[Avail_Evenings] = 1;
Update TGT Set TGT.[Avail_Nights] = SRC.[Avail_Nights] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Avail_Nights] is null Or TGT.[Avail_Nights] = 0) and SRC.[Avail_Nights] = 1;
Update TGT Set TGT.[Avail_WeekDays] = SRC.[Avail_WeekDays] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Avail_WeekDays] is null Or TGT.[Avail_WeekDays] = 0) and SRC.[Avail_WeekDays] = 1;
Update TGT Set TGT.[Avail_Weekends] = SRC.[Avail_Weekends] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Avail_Weekends] is null Or TGT.[Avail_Weekends] = 0) and SRC.[Avail_Weekends] = 1;
Update TGT Set TGT.[Avail_Flexible] = SRC.[Avail_Flexible] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Avail_Flexible] is null Or TGT.[Avail_Flexible] = 0) and SRC.[Avail_Flexible] = 1;
Update TGT Set TGT.[Avail_8Hours] = SRC.[Avail_8Hours] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Avail_8Hours] is null Or TGT.[Avail_8Hours] = 0) and SRC.[Avail_8Hours] = 1;
Update TGT Set TGT.[Avail_12Hours] = SRC.[Avail_12Hours] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Avail_12Hours] is null Or TGT.[Avail_12Hours] = 0) and SRC.[Avail_12Hours] = 1;
Update TGT Set TGT.[Avail_Other] = SRC.[Avail_Other] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Avail_Other] is null Or TGT.[Avail_Other] = 0) and SRC.[Avail_Other] = 1;
Update TGT Set TGT.[Avail_DaysPerWeek] = SRC.[Avail_DaysPerWeek] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Avail_DaysPerWeek] is null and SRC.[Avail_DaysPerWeek] is not null;
Update TGT Set TGT.[Avail_FullTime] = SRC.[Avail_FullTime] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Avail_FullTime] is null Or TGT.[Avail_FullTime] = 0) and SRC.[Avail_FullTime] = 1;
Update TGT Set TGT.[Avail_Date] = SRC.[Avail_Date] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Avail_Date] is null And SRC.[Avail_Date] is not null) OR (TGT.[Avail_Date] is not null And SRC.[Avail_Date] is not null And TGT.[Avail_Date] < SRC.[Avail_Date]));
Update TGT Set TGT.[Recruit_FacilityID_Initial] = SRC.[Recruit_FacilityID_Initial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Recruit_FacilityID_Initial] is null and SRC.[Recruit_FacilityID_Initial] is not null;
Update TGT Set TGT.[Recruit_FacilityID_Date] = SRC.[Recruit_FacilityID_Date] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Recruit_FacilityID_Date] is null And SRC.[Recruit_FacilityID_Date] is not null) OR (TGT.[Recruit_FacilityID_Date] is not null And SRC.[Recruit_FacilityID_Date] is not null And TGT.[Recruit_FacilityID_Date] < SRC.[Recruit_FacilityID_Date]));
Update TGT Set TGT.[Recruit_FacilityID] = SRC.[Recruit_FacilityID] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Recruit_FacilityID] is null and SRC.[Recruit_FacilityID] is not null;
Update TGT Set TGT.[Recruit_FacilityID_Initial_Entered] = SRC.[Recruit_FacilityID_Initial_Entered] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Recruit_FacilityID_Initial_Entered] is null and SRC.[Recruit_FacilityID_Initial_Entered] is not null;
Update TGT Set TGT.[Email_Status] = SRC.[Email_Status] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Email_Status] is null and SRC.[Email_Status] is not null;
Update TGT Set TGT.[Recruitment_EmployeeID] = SRC.[Recruitment_EmployeeID] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Recruitment_EmployeeID] is null and SRC.[Recruitment_EmployeeID] is not null;
Update TGT Set TGT.[FileCompleted] = SRC.[FileCompleted] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[FileCompleted] is null Or TGT.[FileCompleted] = 0) and SRC.[FileCompleted] = 1;
Update TGT Set TGT.[Note] = SRC.[Note] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Note] is null and SRC.[Note] is not null;
Update TGT Set TGT.[Print_Label] = SRC.[Print_Label] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Print_Label] is null Or TGT.[Print_Label] = 0) and SRC.[Print_Label] = 1;
Update TGT Set TGT.[Initial_Deleted] = SRC.[Initial_Deleted] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Initial_Deleted] is null and SRC.[Initial_Deleted] is not null;
Update TGT Set TGT.[Reason_Deleted] = SRC.[Reason_Deleted] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Reason_Deleted] is null and SRC.[Reason_Deleted] is not null;
Update TGT Set TGT.[DateEntered] = SRC.[DateEntered] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[DateEntered] is null And SRC.[DateEntered] is not null) OR (TGT.[DateEntered] is not null And SRC.[DateEntered] is not null And TGT.[DateEntered] < SRC.[DateEntered]));
Update TGT Set TGT.[HasSpecialtyLicenses_PCA] = SRC.[HasSpecialtyLicenses_PCA] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HasSpecialtyLicenses_PCA] is null Or TGT.[HasSpecialtyLicenses_PCA] = 0) and SRC.[HasSpecialtyLicenses_PCA] = 1;
Update TGT Set TGT.[HasSpecialtyLicenses_ORT] = SRC.[HasSpecialtyLicenses_ORT] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HasSpecialtyLicenses_ORT] is null Or TGT.[HasSpecialtyLicenses_ORT] = 0) and SRC.[HasSpecialtyLicenses_ORT] = 1;
Update TGT Set TGT.[HasSpecialtyLicenses_NT] = SRC.[HasSpecialtyLicenses_NT] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HasSpecialtyLicenses_NT] is null Or TGT.[HasSpecialtyLicenses_NT] = 0) and SRC.[HasSpecialtyLicenses_NT] = 1;
Update TGT Set TGT.[HasSpecialtyLicenses_HHA] = SRC.[HasSpecialtyLicenses_HHA] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HasSpecialtyLicenses_HHA] is null Or TGT.[HasSpecialtyLicenses_HHA] = 0) and SRC.[HasSpecialtyLicenses_HHA] = 1;
Update TGT Set TGT.[HasSpecialtyLicenses_PCT] = SRC.[HasSpecialtyLicenses_PCT] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HasSpecialtyLicenses_PCT] is null Or TGT.[HasSpecialtyLicenses_PCT] = 0) and SRC.[HasSpecialtyLicenses_PCT] = 1;
Update TGT Set TGT.[Streamline_Not_Received_Card] = SRC.[Streamline_Not_Received_Card] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Streamline_Not_Received_Card] is null Or TGT.[Streamline_Not_Received_Card] = 0) and SRC.[Streamline_Not_Received_Card] = 1;
Update TGT Set TGT.[SelfScheduled] = SRC.[SelfScheduled] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[SelfScheduled] is null Or TGT.[SelfScheduled] = 0) and SRC.[SelfScheduled] = 1;
Update TGT Set TGT.[Bilingual] = SRC.[Bilingual] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Bilingual] is null and SRC.[Bilingual] is not null;
Update TGT Set TGT.[AvailibilityPDaysOld] = SRC.[AvailibilityPDaysOld] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[AvailibilityPDaysOld] is null and SRC.[AvailibilityPDaysOld] is not null;
Update TGT Set TGT.[AvailibilityPShifts] = SRC.[AvailibilityPShifts] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[AvailibilityPShifts] is null and SRC.[AvailibilityPShifts] is not null;
Update TGT Set TGT.[AvailibilityPDate] = SRC.[AvailibilityPDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[AvailibilityPDate] is null And SRC.[AvailibilityPDate] is not null) OR (TGT.[AvailibilityPDate] is not null And SRC.[AvailibilityPDate] is not null And TGT.[AvailibilityPDate] < SRC.[AvailibilityPDate]));
Update TGT Set TGT.[AvailibilityPDays] = SRC.[AvailibilityPDays] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[AvailibilityPDays] is null and SRC.[AvailibilityPDays] is not null;
Update TGT Set TGT.[CoreMandatory] = SRC.[CoreMandatory] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[CoreMandatory] is null Or TGT.[CoreMandatory] = 0) and SRC.[CoreMandatory] = 1;
Update TGT Set TGT.[CoreMandatoryDate] = SRC.[CoreMandatoryDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[CoreMandatoryDate] is null And SRC.[CoreMandatoryDate] is not null) OR (TGT.[CoreMandatoryDate] is not null And SRC.[CoreMandatoryDate] is not null And TGT.[CoreMandatoryDate] < SRC.[CoreMandatoryDate]));
Update TGT Set TGT.[FileCompleteDate] = SRC.[FileCompleteDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[FileCompleteDate] is null And SRC.[FileCompleteDate] is not null) OR (TGT.[FileCompleteDate] is not null And SRC.[FileCompleteDate] is not null And TGT.[FileCompleteDate] < SRC.[FileCompleteDate]));
Update TGT Set TGT.[FileCompleteInitial] = SRC.[FileCompleteInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[FileCompleteInitial] is null and SRC.[FileCompleteInitial] is not null;
Update TGT Set TGT.[References1Date] = SRC.[References1Date] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[References1Date] is null And SRC.[References1Date] is not null) OR (TGT.[References1Date] is not null And SRC.[References1Date] is not null And TGT.[References1Date] < SRC.[References1Date]));
Update TGT Set TGT.[References1Initial] = SRC.[References1Initial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[References1Initial] is null and SRC.[References1Initial] is not null;
Update TGT Set TGT.[References2Date] = SRC.[References2Date] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[References2Date] is null And SRC.[References2Date] is not null) OR (TGT.[References2Date] is not null And SRC.[References2Date] is not null And TGT.[References2Date] < SRC.[References2Date]));
Update TGT Set TGT.[References2Initial] = SRC.[References2Initial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[References2Initial] is null and SRC.[References2Initial] is not null;
Update TGT Set TGT.[SanctionsDate] = SRC.[SanctionsDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[SanctionsDate] is null And SRC.[SanctionsDate] is not null) OR (TGT.[SanctionsDate] is not null And SRC.[SanctionsDate] is not null And TGT.[SanctionsDate] < SRC.[SanctionsDate]));
Update TGT Set TGT.[SanctionsResults] = SRC.[SanctionsResults] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[SanctionsResults] is null and SRC.[SanctionsResults] is not null;
Update TGT Set TGT.[OIGDate] = SRC.[OIGDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[OIGDate] is null And SRC.[OIGDate] is not null) OR (TGT.[OIGDate] is not null And SRC.[OIGDate] is not null And TGT.[OIGDate] < SRC.[OIGDate]));
Update TGT Set TGT.[OIGResults] = SRC.[OIGResults] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[OIGResults] is null and SRC.[OIGResults] is not null;
Update TGT Set TGT.[HIPAADate] = SRC.[HIPAADate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[HIPAADate] is null And SRC.[HIPAADate] is not null) OR (TGT.[HIPAADate] is not null And SRC.[HIPAADate] is not null And TGT.[HIPAADate] < SRC.[HIPAADate]));
Update TGT Set TGT.[BackgroundCheckDate] = SRC.[BackgroundCheckDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[BackgroundCheckDate] is null And SRC.[BackgroundCheckDate] is not null) OR (TGT.[BackgroundCheckDate] is not null And SRC.[BackgroundCheckDate] is not null And TGT.[BackgroundCheckDate] < SRC.[BackgroundCheckDate]));
Update TGT Set TGT.[EPLSDate] = SRC.[EPLSDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[EPLSDate] is null And SRC.[EPLSDate] is not null) OR (TGT.[EPLSDate] is not null And SRC.[EPLSDate] is not null And TGT.[EPLSDate] < SRC.[EPLSDate]));
Update TGT Set TGT.[EPLSResults] = SRC.[EPLSResults] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[EPLSResults] is null and SRC.[EPLSResults] is not null;
Update TGT Set TGT.[ts] = SRC.[ts] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ts] is null and SRC.[ts] is not null;
Update TGT Set TGT.[HomeCare] = SRC.[HomeCare] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HomeCare] is null Or TGT.[HomeCare] = 0) and SRC.[HomeCare] = 1;
Update TGT Set TGT.[BackgroundCheckResults] = SRC.[BackgroundCheckResults] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[BackgroundCheckResults] is null and SRC.[BackgroundCheckResults] is not null;
Update TGT Set TGT.[ChauncySanctionsDate] = SRC.[ChauncySanctionsDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ChauncySanctionsDate] is null And SRC.[ChauncySanctionsDate] is not null) OR (TGT.[ChauncySanctionsDate] is not null And SRC.[ChauncySanctionsDate] is not null And TGT.[ChauncySanctionsDate] < SRC.[ChauncySanctionsDate]));
Update TGT Set TGT.[EducationVerified] = SRC.[EducationVerified] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[EducationVerified] is null Or TGT.[EducationVerified] = 0) and SRC.[EducationVerified] = 1;
Update TGT Set TGT.[WGIDStatus] = SRC.[WGIDStatus] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[WGIDStatus] is null and SRC.[WGIDStatus] is not null;
Update TGT Set TGT.[BackgroundAgency] = SRC.[BackgroundAgency] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[BackgroundAgency] is null and SRC.[BackgroundAgency] is not null;
Update TGT Set TGT.[BackgroundCheckConsent] = SRC.[BackgroundCheckConsent] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[BackgroundCheckConsent] is null Or TGT.[BackgroundCheckConsent] = 0) and SRC.[BackgroundCheckConsent] = 1;
Update TGT Set TGT.[ChauncySanctionsResults] = SRC.[ChauncySanctionsResults] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ChauncySanctionsResults] is null and SRC.[ChauncySanctionsResults] is not null;
Update TGT Set TGT.[veteran] = SRC.[veteran] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[veteran] is null Or TGT.[veteran] = 0) and SRC.[veteran] = 1;
Update TGT Set TGT.[NPI] = SRC.[NPI] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[NPI] is null and SRC.[NPI] is not null;
Update TGT Set TGT.[Performance_Eval_Comp_YN] = SRC.[Performance_Eval_Comp_YN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[Performance_Eval_Comp_YN] is null Or TGT.[Performance_Eval_Comp_YN] = 0) and SRC.[Performance_Eval_Comp_YN] = 1;
Update TGT Set TGT.[LocalContract] = SRC.[LocalContract] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[LocalContract] is null Or TGT.[LocalContract] = 0) and SRC.[LocalContract] = 1;
Update TGT Set TGT.[VolSelfID] = SRC.[VolSelfID] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[VolSelfID] is null Or TGT.[VolSelfID] = 0) and SRC.[VolSelfID] = 1;
Update TGT Set TGT.[SMSProvider] = SRC.[SMSProvider] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[SMSProvider] is null and SRC.[SMSProvider] is not null;
Update TGT Set TGT.[IntrestedinVAFacilities] = SRC.[IntrestedinVAFacilities] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[IntrestedinVAFacilities] is null Or TGT.[IntrestedinVAFacilities] = 0) and SRC.[IntrestedinVAFacilities] = 1;
Update TGT Set TGT.[NotAvailChecked] = SRC.[NotAvailChecked] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[NotAvailChecked] is null Or TGT.[NotAvailChecked] = 0) and SRC.[NotAvailChecked] = 1;
Update TGT Set TGT.[NotAvailUntil] = SRC.[NotAvailUntil] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[NotAvailUntil] is null And SRC.[NotAvailUntil] is not null) OR (TGT.[NotAvailUntil] is not null And SRC.[NotAvailUntil] is not null And TGT.[NotAvailUntil] < SRC.[NotAvailUntil]));
Update TGT Set TGT.[NotSendEmail] = SRC.[NotSendEmail] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[NotSendEmail] is null Or TGT.[NotSendEmail] = 0) and SRC.[NotSendEmail] = 1;
Update TGT Set TGT.[NotSendTextMsg] = SRC.[NotSendTextMsg] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[NotSendTextMsg] is null Or TGT.[NotSendTextMsg] = 0) and SRC.[NotSendTextMsg] = 1;
Update TGT Set TGT.[WGDrugScreenDate] = SRC.[WGDrugScreenDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[WGDrugScreenDate] is null And SRC.[WGDrugScreenDate] is not null) OR (TGT.[WGDrugScreenDate] is not null And SRC.[WGDrugScreenDate] is not null And TGT.[WGDrugScreenDate] < SRC.[WGDrugScreenDate]));
Update TGT Set TGT.[MaskFitTest] = SRC.[MaskFitTest] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[MaskFitTest] is null Or TGT.[MaskFitTest] = 0) and SRC.[MaskFitTest] = 1;
Update TGT Set TGT.[FluShutDate] = SRC.[FluShutDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[FluShutDate] is null And SRC.[FluShutDate] is not null) OR (TGT.[FluShutDate] is not null And SRC.[FluShutDate] is not null And TGT.[FluShutDate] < SRC.[FluShutDate]));
Update TGT Set TGT.[OrientationDocumentation] = SRC.[OrientationDocumentation] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[OrientationDocumentation] is null Or TGT.[OrientationDocumentation] = 0) and SRC.[OrientationDocumentation] = 1;
Update TGT Set TGT.[ApplicationDate] = SRC.[ApplicationDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ApplicationDate] is null And SRC.[ApplicationDate] is not null) OR (TGT.[ApplicationDate] is not null And SRC.[ApplicationDate] is not null And TGT.[ApplicationDate] < SRC.[ApplicationDate]));
Update TGT Set TGT.[H1n1] = SRC.[H1n1] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[H1n1] is null And SRC.[H1n1] is not null) OR (TGT.[H1n1] is not null And SRC.[H1n1] is not null And TGT.[H1n1] < SRC.[H1n1]));
Update TGT Set TGT.[MaskFitTestDate] = SRC.[MaskFitTestDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[MaskFitTestDate] is null And SRC.[MaskFitTestDate] is not null) OR (TGT.[MaskFitTestDate] is not null And SRC.[MaskFitTestDate] is not null And TGT.[MaskFitTestDate] < SRC.[MaskFitTestDate]));
Update TGT Set TGT.[FluExempt] = SRC.[FluExempt] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[FluExempt] is null And SRC.[FluExempt] is not null) OR (TGT.[FluExempt] is not null And SRC.[FluExempt] is not null And TGT.[FluExempt] < SRC.[FluExempt]));
Update TGT Set TGT.[SkillsChecklistScore] = SRC.[SkillsChecklistScore] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[SkillsChecklistScore] is null and SRC.[SkillsChecklistScore] is not null;
Update TGT Set TGT.[MeaslesRubeolaLabReports] = SRC.[MeaslesRubeolaLabReports] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[MeaslesRubeolaLabReports] is null Or TGT.[MeaslesRubeolaLabReports] = 0) and SRC.[MeaslesRubeolaLabReports] = 1;
Update TGT Set TGT.[MumpsLabReports] = SRC.[MumpsLabReports] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[MumpsLabReports] is null Or TGT.[MumpsLabReports] = 0) and SRC.[MumpsLabReports] = 1;
Update TGT Set TGT.[RubellaLabReports] = SRC.[RubellaLabReports] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[RubellaLabReports] is null Or TGT.[RubellaLabReports] = 0) and SRC.[RubellaLabReports] = 1;
Update TGT Set TGT.[VaricellaLabReports] = SRC.[VaricellaLabReports] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[VaricellaLabReports] is null Or TGT.[VaricellaLabReports] = 0) and SRC.[VaricellaLabReports] = 1;
Update TGT Set TGT.[LicenseNumSignedYN] = SRC.[LicenseNumSignedYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[LicenseNumSignedYN] is null Or TGT.[LicenseNumSignedYN] = 0) and SRC.[LicenseNumSignedYN] = 1;
Update TGT Set TGT.[FacilityCompleted] = SRC.[FacilityCompleted] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[FacilityCompleted] is null Or TGT.[FacilityCompleted] = 0) and SRC.[FacilityCompleted] = 1;
Update TGT Set TGT.[FacilityCompletedDate] = SRC.[FacilityCompletedDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[FacilityCompletedDate] is null And SRC.[FacilityCompletedDate] is not null) OR (TGT.[FacilityCompletedDate] is not null And SRC.[FacilityCompletedDate] is not null And TGT.[FacilityCompletedDate] < SRC.[FacilityCompletedDate]));
Update TGT Set TGT.[FacilityCompletedInitial] = SRC.[FacilityCompletedInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[FacilityCompletedInitial] is null and SRC.[FacilityCompletedInitial] is not null;
Update TGT Set TGT.[SMSStatus] = SRC.[SMSStatus] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[SMSStatus] is null and SRC.[SMSStatus] is not null;
Update TGT Set TGT.[EmailAddressInvalid] = SRC.[EmailAddressInvalid] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[EmailAddressInvalid] is null and SRC.[EmailAddressInvalid] is not null;
Update TGT Set TGT.[EmailByPhoneYN] = SRC.[EmailByPhoneYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[EmailByPhoneYN] is null Or TGT.[EmailByPhoneYN] = 0) and SRC.[EmailByPhoneYN] = 1;
Update TGT Set TGT.[OMIGDate] = SRC.[OMIGDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[OMIGDate] is null And SRC.[OMIGDate] is not null) OR (TGT.[OMIGDate] is not null And SRC.[OMIGDate] is not null And TGT.[OMIGDate] < SRC.[OMIGDate]));
Update TGT Set TGT.[OMIGResults] = SRC.[OMIGResults] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[OMIGResults] is null and SRC.[OMIGResults] is not null;
Update TGT Set TGT.[TBQDate] = SRC.[TBQDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[TBQDate] is null And SRC.[TBQDate] is not null) OR (TGT.[TBQDate] is not null And SRC.[TBQDate] is not null And TGT.[TBQDate] < SRC.[TBQDate]));
Update TGT Set TGT.[TBQResults] = SRC.[TBQResults] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[TBQResults] is null and SRC.[TBQResults] is not null;
Update TGT Set TGT.[--NotForHomeCare--] = SRC.[--NotForHomeCare--] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[--NotForHomeCare--] is null Or TGT.[--NotForHomeCare--] = 0) and SRC.[--NotForHomeCare--] = 1;
Update TGT Set TGT.[Original_Address] = SRC.[Original_Address] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Original_Address] is null and SRC.[Original_Address] is not null;
Update TGT Set TGT.[Original_City] = SRC.[Original_City] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Original_City] is null and SRC.[Original_City] is not null;
Update TGT Set TGT.[Original_State] = SRC.[Original_State] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Original_State] is null and SRC.[Original_State] is not null;
Update TGT Set TGT.[Original_Zip] = SRC.[Original_Zip] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Original_Zip] is null and SRC.[Original_Zip] is not null;
Update TGT Set TGT.[ProofOfOriginalAddress] = SRC.[ProofOfOriginalAddress] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ProofOfOriginalAddress] is null and SRC.[ProofOfOriginalAddress] is not null;
Update TGT Set TGT.[ProofOfOriginalAddressDate] = SRC.[ProofOfOriginalAddressDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ProofOfOriginalAddressDate] is null And SRC.[ProofOfOriginalAddressDate] is not null) OR (TGT.[ProofOfOriginalAddressDate] is not null And SRC.[ProofOfOriginalAddressDate] is not null And TGT.[ProofOfOriginalAddressDate] < SRC.[ProofOfOriginalAddressDate]));
Update TGT Set TGT.[ProofOfOriginalAddressInitial] = SRC.[ProofOfOriginalAddressInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ProofOfOriginalAddressInitial] is null and SRC.[ProofOfOriginalAddressInitial] is not null;
Update TGT Set TGT.[CriminalPerApp] = SRC.[CriminalPerApp] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[CriminalPerApp] is null Or TGT.[CriminalPerApp] = 0) and SRC.[CriminalPerApp] = 1;
Update TGT Set TGT.[JcahoColor] = SRC.[JcahoColor] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[JcahoColor] is null and SRC.[JcahoColor] is not null;
Update TGT Set TGT.[JcahoColorDate] = SRC.[JcahoColorDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[JcahoColorDate] is null And SRC.[JcahoColorDate] is not null) OR (TGT.[JcahoColorDate] is not null And SRC.[JcahoColorDate] is not null And TGT.[JcahoColorDate] < SRC.[JcahoColorDate]));
Update TGT Set TGT.[JcahoDueDate] = SRC.[JcahoDueDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[JcahoDueDate] is null And SRC.[JcahoDueDate] is not null) OR (TGT.[JcahoDueDate] is not null And SRC.[JcahoDueDate] is not null And TGT.[JcahoDueDate] < SRC.[JcahoDueDate]));
Update TGT Set TGT.[BSN_YN] = SRC.[BSN_YN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[BSN_YN] is null Or TGT.[BSN_YN] = 0) and SRC.[BSN_YN] = 1;
Update TGT Set TGT.[BclsSignedYN] = SRC.[BclsSignedYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[BclsSignedYN] is null Or TGT.[BclsSignedYN] = 0) and SRC.[BclsSignedYN] = 1;
Update TGT Set TGT.[AclsSignedYN] = SRC.[AclsSignedYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[AclsSignedYN] is null Or TGT.[AclsSignedYN] = 0) and SRC.[AclsSignedYN] = 1;
Update TGT Set TGT.[NalsSignedYN] = SRC.[NalsSignedYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[NalsSignedYN] is null Or TGT.[NalsSignedYN] = 0) and SRC.[NalsSignedYN] = 1;
Update TGT Set TGT.[PalsSignedYN] = SRC.[PalsSignedYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[PalsSignedYN] is null Or TGT.[PalsSignedYN] = 0) and SRC.[PalsSignedYN] = 1;
Update TGT Set TGT.[LSFormYN] = SRC.[LSFormYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[LSFormYN] is null Or TGT.[LSFormYN] = 0) and SRC.[LSFormYN] = 1;
Update TGT Set TGT.[ResumeYNDate] = SRC.[ResumeYNDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ResumeYNDate] is null And SRC.[ResumeYNDate] is not null) OR (TGT.[ResumeYNDate] is not null And SRC.[ResumeYNDate] is not null And TGT.[ResumeYNDate] < SRC.[ResumeYNDate]));
Update TGT Set TGT.[FacRate] = SRC.[FacRate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[FacRate] is null and SRC.[FacRate] is not null;
Update TGT Set TGT.[FacRateDate] = SRC.[FacRateDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[FacRateDate] is null And SRC.[FacRateDate] is not null) OR (TGT.[FacRateDate] is not null And SRC.[FacRateDate] is not null And TGT.[FacRateDate] < SRC.[FacRateDate]));
Update TGT Set TGT.[FacRateInit] = SRC.[FacRateInit] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[FacRateInit] is null and SRC.[FacRateInit] is not null;
Update TGT Set TGT.[LSOnApplyingDate] = SRC.[LSOnApplyingDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[LSOnApplyingDate] is null And SRC.[LSOnApplyingDate] is not null) OR (TGT.[LSOnApplyingDate] is not null And SRC.[LSOnApplyingDate] is not null And TGT.[LSOnApplyingDate] < SRC.[LSOnApplyingDate]));
Update TGT Set TGT.[Title2] = SRC.[Title2] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Title2] is null and SRC.[Title2] is not null;
Update TGT Set TGT.[SkillsChecklistScore2] = SRC.[SkillsChecklistScore2] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[SkillsChecklistScore2] is null and SRC.[SkillsChecklistScore2] is not null;
Update TGT Set TGT.[Degree] = SRC.[Degree] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Degree] is null and SRC.[Degree] is not null;
Update TGT Set TGT.[SMSProviderInvalid] = SRC.[SMSProviderInvalid] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[SMSProviderInvalid] is null and SRC.[SMSProviderInvalid] is not null;
Update TGT Set TGT.[LicenseState] = SRC.[LicenseState] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[LicenseState] is null and SRC.[LicenseState] is not null;
Update TGT Set TGT.[MalLevelOK] = SRC.[MalLevelOK] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[MalLevelOK] is null Or TGT.[MalLevelOK] = 0) and SRC.[MalLevelOK] = 1;
Update TGT Set TGT.[NotSendMassEmail] = SRC.[NotSendMassEmail] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[NotSendMassEmail] is null Or TGT.[NotSendMassEmail] = 0) and SRC.[NotSendMassEmail] = 1;
Update TGT Set TGT.[NotSendMassTextMsg] = SRC.[NotSendMassTextMsg] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[NotSendMassTextMsg] is null Or TGT.[NotSendMassTextMsg] = 0) and SRC.[NotSendMassTextMsg] = 1;
Update TGT Set TGT.[AttestationFormDate] = SRC.[AttestationFormDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[AttestationFormDate] is null And SRC.[AttestationFormDate] is not null) OR (TGT.[AttestationFormDate] is not null And SRC.[AttestationFormDate] is not null And TGT.[AttestationFormDate] < SRC.[AttestationFormDate]));
Update TGT Set TGT.[AttestationFormUploadedDate] = SRC.[AttestationFormUploadedDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[AttestationFormUploadedDate] is null And SRC.[AttestationFormUploadedDate] is not null) OR (TGT.[AttestationFormUploadedDate] is not null And SRC.[AttestationFormUploadedDate] is not null And TGT.[AttestationFormUploadedDate] < SRC.[AttestationFormUploadedDate]));
Update TGT Set TGT.[AttestationFormSigned] = SRC.[AttestationFormSigned] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[AttestationFormSigned] is null Or TGT.[AttestationFormSigned] = 0) and SRC.[AttestationFormSigned] = 1;
Update TGT Set TGT.[CorporateCompliancePolicyDate] = SRC.[CorporateCompliancePolicyDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[CorporateCompliancePolicyDate] is null And SRC.[CorporateCompliancePolicyDate] is not null) OR (TGT.[CorporateCompliancePolicyDate] is not null And SRC.[CorporateCompliancePolicyDate] is not null And TGT.[CorporateCompliancePolicyDate] < SRC.[CorporateCompliancePolicyDate]));
Update TGT Set TGT.[RN_LPN_HC] = SRC.[RN_LPN_HC] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[RN_LPN_HC] is null And SRC.[RN_LPN_HC] is not null) OR (TGT.[RN_LPN_HC] is not null And SRC.[RN_LPN_HC] is not null And TGT.[RN_LPN_HC] < SRC.[RN_LPN_HC]));
Update TGT Set TGT.[SSAYN] = SRC.[SSAYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[SSAYN] is null Or TGT.[SSAYN] = 0) and SRC.[SSAYN] = 1;
Update TGT Set TGT.[HomeCareExamsDueDate] = SRC.[HomeCareExamsDueDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[HomeCareExamsDueDate] is null And SRC.[HomeCareExamsDueDate] is not null) OR (TGT.[HomeCareExamsDueDate] is not null And SRC.[HomeCareExamsDueDate] is not null And TGT.[HomeCareExamsDueDate] < SRC.[HomeCareExamsDueDate]));
Update TGT Set TGT.[EmailLastVerifiedDate] = SRC.[EmailLastVerifiedDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[EmailLastVerifiedDate] is null And SRC.[EmailLastVerifiedDate] is not null) OR (TGT.[EmailLastVerifiedDate] is not null And SRC.[EmailLastVerifiedDate] is not null And TGT.[EmailLastVerifiedDate] < SRC.[EmailLastVerifiedDate]));
Update TGT Set TGT.[SMSProviderLastVerifiedDate] = SRC.[SMSProviderLastVerifiedDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[SMSProviderLastVerifiedDate] is null And SRC.[SMSProviderLastVerifiedDate] is not null) OR (TGT.[SMSProviderLastVerifiedDate] is not null And SRC.[SMSProviderLastVerifiedDate] is not null And TGT.[SMSProviderLastVerifiedDate] < SRC.[SMSProviderLastVerifiedDate]));
Update TGT Set TGT.[NSOSDate] = SRC.[NSOSDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[NSOSDate] is null And SRC.[NSOSDate] is not null) OR (TGT.[NSOSDate] is not null And SRC.[NSOSDate] is not null And TGT.[NSOSDate] < SRC.[NSOSDate]));
Update TGT Set TGT.[NSOSRes] = SRC.[NSOSRes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[NSOSRes] is null and SRC.[NSOSRes] is not null;
Update TGT Set TGT.[EmailLastVerifiedInitial] = SRC.[EmailLastVerifiedInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[EmailLastVerifiedInitial] is null and SRC.[EmailLastVerifiedInitial] is not null;
Update TGT Set TGT.[WGRecordsYN] = SRC.[WGRecordsYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[WGRecordsYN] is null Or TGT.[WGRecordsYN] = 0) and SRC.[WGRecordsYN] = 1;
Update TGT Set TGT.[WGRecordsDate] = SRC.[WGRecordsDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[WGRecordsDate] is null And SRC.[WGRecordsDate] is not null) OR (TGT.[WGRecordsDate] is not null And SRC.[WGRecordsDate] is not null And TGT.[WGRecordsDate] < SRC.[WGRecordsDate]));
Update TGT Set TGT.[WGRecordsInitial] = SRC.[WGRecordsInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[WGRecordsInitial] is null and SRC.[WGRecordsInitial] is not null;
Update TGT Set TGT.[VentTrainingClass] = SRC.[VentTrainingClass] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[VentTrainingClass] is null and SRC.[VentTrainingClass] is not null;
Update TGT Set TGT.[VentCertificateYN] = SRC.[VentCertificateYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[VentCertificateYN] is null Or TGT.[VentCertificateYN] = 0) and SRC.[VentCertificateYN] = 1;
Update TGT Set TGT.[VentSupervision1YN] = SRC.[VentSupervision1YN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[VentSupervision1YN] is null Or TGT.[VentSupervision1YN] = 0) and SRC.[VentSupervision1YN] = 1;
Update TGT Set TGT.[VentSupervision2YN] = SRC.[VentSupervision2YN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[VentSupervision2YN] is null Or TGT.[VentSupervision2YN] = 0) and SRC.[VentSupervision2YN] = 1;
Update TGT Set TGT.[VentSupervision3YN] = SRC.[VentSupervision3YN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[VentSupervision3YN] is null Or TGT.[VentSupervision3YN] = 0) and SRC.[VentSupervision3YN] = 1;
Update TGT Set TGT.[FluAttestationFormDate] = SRC.[FluAttestationFormDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[FluAttestationFormDate] is null And SRC.[FluAttestationFormDate] is not null) OR (TGT.[FluAttestationFormDate] is not null And SRC.[FluAttestationFormDate] is not null And TGT.[FluAttestationFormDate] < SRC.[FluAttestationFormDate]));
Update TGT Set TGT.[FluAttestationFormUploadedDate] = SRC.[FluAttestationFormUploadedDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[FluAttestationFormUploadedDate] is null And SRC.[FluAttestationFormUploadedDate] is not null) OR (TGT.[FluAttestationFormUploadedDate] is not null And SRC.[FluAttestationFormUploadedDate] is not null And TGT.[FluAttestationFormUploadedDate] < SRC.[FluAttestationFormUploadedDate]));
Update TGT Set TGT.[BclsLetterDate] = SRC.[BclsLetterDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[BclsLetterDate] is null And SRC.[BclsLetterDate] is not null) OR (TGT.[BclsLetterDate] is not null And SRC.[BclsLetterDate] is not null And TGT.[BclsLetterDate] < SRC.[BclsLetterDate]));
Update TGT Set TGT.[AclsLetterDate] = SRC.[AclsLetterDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[AclsLetterDate] is null And SRC.[AclsLetterDate] is not null) OR (TGT.[AclsLetterDate] is not null And SRC.[AclsLetterDate] is not null And TGT.[AclsLetterDate] < SRC.[AclsLetterDate]));
Update TGT Set TGT.[NalsLetterDate] = SRC.[NalsLetterDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[NalsLetterDate] is null And SRC.[NalsLetterDate] is not null) OR (TGT.[NalsLetterDate] is not null And SRC.[NalsLetterDate] is not null And TGT.[NalsLetterDate] < SRC.[NalsLetterDate]));
Update TGT Set TGT.[PalsLetterDate] = SRC.[PalsLetterDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[PalsLetterDate] is null And SRC.[PalsLetterDate] is not null) OR (TGT.[PalsLetterDate] is not null And SRC.[PalsLetterDate] is not null And TGT.[PalsLetterDate] < SRC.[PalsLetterDate]));
Update TGT Set TGT.[W4Date] = SRC.[W4Date] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[W4Date] is null And SRC.[W4Date] is not null) OR (TGT.[W4Date] is not null And SRC.[W4Date] is not null And TGT.[W4Date] < SRC.[W4Date]));
Update TGT Set TGT.[EthnicGroup] = SRC.[EthnicGroup] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[EthnicGroup] is null and SRC.[EthnicGroup] is not null;
Update TGT Set TGT.[SizeModel] = SRC.[SizeModel] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[SizeModel] is null and SRC.[SizeModel] is not null;
Update TGT Set TGT.[ExpectedGraduation] = SRC.[ExpectedGraduation] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ExpectedGraduation] is null and SRC.[ExpectedGraduation] is not null;
Update TGT Set TGT.[ExpectedGraduationDate] = SRC.[ExpectedGraduationDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ExpectedGraduationDate] is null And SRC.[ExpectedGraduationDate] is not null) OR (TGT.[ExpectedGraduationDate] is not null And SRC.[ExpectedGraduationDate] is not null And TGT.[ExpectedGraduationDate] < SRC.[ExpectedGraduationDate]));
Update TGT Set TGT.[EVerifyYN] = SRC.[EVerifyYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[EVerifyYN] is null Or TGT.[EVerifyYN] = 0) and SRC.[EVerifyYN] = 1;
Update TGT Set TGT.[EmailVerifiedByVendorDate] = SRC.[EmailVerifiedByVendorDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[EmailVerifiedByVendorDate] is null And SRC.[EmailVerifiedByVendorDate] is not null) OR (TGT.[EmailVerifiedByVendorDate] is not null And SRC.[EmailVerifiedByVendorDate] is not null And TGT.[EmailVerifiedByVendorDate] < SRC.[EmailVerifiedByVendorDate]));
Update TGT Set TGT.[MedicalClearanceYN] = SRC.[MedicalClearanceYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[MedicalClearanceYN] is null Or TGT.[MedicalClearanceYN] = 0) and SRC.[MedicalClearanceYN] = 1;
Update TGT Set TGT.[MedicalClearanceFacility] = SRC.[MedicalClearanceFacility] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[MedicalClearanceFacility] is null and SRC.[MedicalClearanceFacility] is not null;
Update TGT Set TGT.[InsuranceStatus] = SRC.[InsuranceStatus] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[InsuranceStatus] is null and SRC.[InsuranceStatus] is not null;
Update TGT Set TGT.[BestTimeToReach] = SRC.[BestTimeToReach] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[BestTimeToReach] is null and SRC.[BestTimeToReach] is not null;
Update TGT Set TGT.[ReferredByName] = SRC.[ReferredByName] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ReferredByName] is null and SRC.[ReferredByName] is not null;
Update TGT Set TGT.[TaxCredit8850YN] = SRC.[TaxCredit8850YN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[TaxCredit8850YN] is null Or TGT.[TaxCredit8850YN] = 0) and SRC.[TaxCredit8850YN] = 1;
Update TGT Set TGT.[TaxCredit8850] = SRC.[TaxCredit8850] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[TaxCredit8850] is null and SRC.[TaxCredit8850] is not null;
Update TGT Set TGT.[ResumeYNInitial] = SRC.[ResumeYNInitial] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ResumeYNInitial] is null and SRC.[ResumeYNInitial] is not null;
Update TGT Set TGT.[COIDate] = SRC.[COIDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[COIDate] is null And SRC.[COIDate] is not null) OR (TGT.[COIDate] is not null And SRC.[COIDate] is not null And TGT.[COIDate] < SRC.[COIDate]));
Update TGT Set TGT.[OrientationDocumentationFacility] = SRC.[OrientationDocumentationFacility] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[OrientationDocumentationFacility] is null and SRC.[OrientationDocumentationFacility] is not null;
Update TGT Set TGT.[CPIDate] = SRC.[CPIDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[CPIDate] is null And SRC.[CPIDate] is not null) OR (TGT.[CPIDate] is not null And SRC.[CPIDate] is not null And TGT.[CPIDate] < SRC.[CPIDate]));
Update TGT Set TGT.[HepBTiter] = SRC.[HepBTiter] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HepBTiter] is null Or TGT.[HepBTiter] = 0) and SRC.[HepBTiter] = 1;
Update TGT Set TGT.[HepBLabReports] = SRC.[HepBLabReports] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HepBLabReports] is null Or TGT.[HepBLabReports] = 0) and SRC.[HepBLabReports] = 1;
Update TGT Set TGT.[HepBRes] = SRC.[HepBRes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[HepBRes] is null and SRC.[HepBRes] is not null;
Update TGT Set TGT.[OmigOigSamDate] = SRC.[OmigOigSamDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[OmigOigSamDate] is null And SRC.[OmigOigSamDate] is not null) OR (TGT.[OmigOigSamDate] is not null And SRC.[OmigOigSamDate] is not null And TGT.[OmigOigSamDate] < SRC.[OmigOigSamDate]));
Update TGT Set TGT.[OmigOigSamRes] = SRC.[OmigOigSamRes] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[OmigOigSamRes] is null and SRC.[OmigOigSamRes] is not null;
Update TGT Set TGT.[DriveYN] = SRC.[DriveYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[DriveYN] is null and SRC.[DriveYN] is not null;
Update TGT Set TGT.[Email2] = SRC.[Email2] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Email2] is null and SRC.[Email2] is not null;
Update TGT Set TGT.[Email2VerifiedByVendorDate] = SRC.[Email2VerifiedByVendorDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[Email2VerifiedByVendorDate] is null And SRC.[Email2VerifiedByVendorDate] is not null) OR (TGT.[Email2VerifiedByVendorDate] is not null And SRC.[Email2VerifiedByVendorDate] is not null And TGT.[Email2VerifiedByVendorDate] < SRC.[Email2VerifiedByVendorDate]));
Update TGT Set TGT.[Email2_Status] = SRC.[Email2_Status] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Email2_Status] is null and SRC.[Email2_Status] is not null;
Update TGT Set TGT.[ORTVerificationDate] = SRC.[ORTVerificationDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[ORTVerificationDate] is null And SRC.[ORTVerificationDate] is not null) OR (TGT.[ORTVerificationDate] is not null And SRC.[ORTVerificationDate] is not null And TGT.[ORTVerificationDate] < SRC.[ORTVerificationDate]));
Update TGT Set TGT.[ORTResults] = SRC.[ORTResults] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[ORTResults] is null and SRC.[ORTResults] is not null;
Update TGT Set TGT.[HepBDeclYN] = SRC.[HepBDeclYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[HepBDeclYN] is null Or TGT.[HepBDeclYN] = 0) and SRC.[HepBDeclYN] = 1;
Update TGT Set TGT.[TBQuantYN] = SRC.[TBQuantYN] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[TBQuantYN] is null Or TGT.[TBQuantYN] = 0) and SRC.[TBQuantYN] = 1;
Update TGT Set TGT.[EverifiedDate] = SRC.[EverifiedDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[EverifiedDate] is null And SRC.[EverifiedDate] is not null) OR (TGT.[EverifiedDate] is not null And SRC.[EverifiedDate] is not null And TGT.[EverifiedDate] < SRC.[EverifiedDate]));
Update TGT Set TGT.[EducationVerifiedDate] = SRC.[EducationVerifiedDate] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[EducationVerifiedDate] is null And SRC.[EducationVerifiedDate] is not null) OR (TGT.[EducationVerifiedDate] is not null And SRC.[EducationVerifiedDate] is not null And TGT.[EducationVerifiedDate] < SRC.[EducationVerifiedDate]));
Update TGT Set TGT.[EducationVerifiedType] = SRC.[EducationVerifiedType] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[EducationVerifiedType] is null and SRC.[EducationVerifiedType] is not null;
Update TGT Set TGT.[SSNSearch] = SRC.[SSNSearch] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And (TGT.[SSNSearch] is null Or TGT.[SSNSearch] = 0) and SRC.[SSNSearch] = 1;
Update TGT Set TGT.[Fingerprint] = SRC.[Fingerprint] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And TGT.[Fingerprint] is null and SRC.[Fingerprint] is not null;
Update TGT Set TGT.[DateExportedBS] = SRC.[DateExportedBS] From Employeestbl TGT inner join Employeestbl SRC on TGT.[Email] = SRC.[Email] Where TGT.SkipImport = 0 And SRC.SkipImport = 1 And ((TGT.[DateExportedBS] is null And SRC.[DateExportedBS] is not null) OR (TGT.[DateExportedBS] is not null And SRC.[DateExportedBS] is not null And TGT.[DateExportedBS] < SRC.[DateExportedBS]));
</textarea>  

## Conclusion
This solution was simple and quick.  While I referred to this as a template soluction I think it is almost an inversion of the typical 'template' configurations you will encounter.  Maybe this would be better described as a __macro__ solution in the traditional sense of expansion/substitution of code.

Although performance wasn't discussed, the table should have a non-unique index on the __Email__ column to optimize the joining operations.
