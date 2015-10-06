
GO
/****** Object:  StoredProcedure [dbo].[ApplySecurity]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[ApplySecurity]
	-- Add the parameters for the stored procedure here
	@pRoleId varchar(50),
	@pUserId int,
	@pScreenId int
AS
BEGIN
	SET NOCOUNT ON; 
	
	select 
	--RP.RoleId as 'RoleId',
	M.id as 'ModuleId',SM.id as 'SubModuleId',s.Id as 'ScreenId', M.ModuleName as 'ModuleName',SM.SubModuleName as 'SubModuleName',S.ScreenName as 'ScreenName',S.action as 'action',
	S.Controller as 'Controller',RP.PrivilegeId as 'PrivilegeId',
	RP.bCreate as 'Create',RP.bView as 'View',RP.bUpdate as 'Update',RP.bDelete as 'Delete',
	M.OrderBy as 'ModuleOrderBy',SM.OrderBy as 'SubModuleOrderBy'
    from T_CFG_ROLESPERMISSION RP,T_CFG_MODULE M,T_CFG_SUBMODULE SM,T_CFG_SCREEN S
    where RP.ModuleId=M.id and RP.SubModuleId=SM.id and RP.ScreenId=S.id and RP.ScreenId=@pScreenId
     and RP.RoleId in(Select s from SplitString(@pRoleId,','))
    
    union
    
    select 
    --UP.UserId as 'UserId',
    UM.id as 'ModuleId',USM.id as 'SubModuleId',Us.Id as 'ScreenId', UM.ModuleName as 'ModuleName',USM.SubModuleName as 'SubModuleName',US.ScreenName as 'ScreenName',US.action as 'action',
    US.Controller as 'Controller',UP.PrivilegeId  as 'PrivilegeId',
    UP.bCreate as 'Create',UP.bView as 'View',UP.bUpdate as 'Update',UP.bDelete as 'Delete', 
    UM.OrderBy as 'ModuleOrderBy',USM.OrderBy as 'SubModuleOrderBy'
    from T_CFG_USERPERMISSION UP,T_CFG_MODULE UM,T_CFG_SUBMODULE USM,T_CFG_SCREEN US 
    where UP.ModuleId=UM.id and UP.SubModuleId=USM.id and UP.ScreenId=US.id and UP.ScreenId=@pScreenId
    and UP.UserId =@pUserId order by ModuleOrderBy asc, SubModuleOrderBy asc
    --in(Select s from SplitString(@pRoleId,','))  
    
    
    
END









GO
/****** Object:  StoredProcedure [dbo].[dontDel]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


CREATE Procedure [dbo].[dontDel]

@pInsitutionId bigint, @pBranchId bigint, @pYearofStudyId bigint,@pAcademicYear bigint,@pFromDate nvarchar(20),@pToDate nvarchar(20)
As Begin
DECLARE @query1 AS NVARCHAR(MAX), @cols1 AS NVARCHAR(MAX)
;With StuInfo(StuTabId,StuName, StuUnivRegNo,StuBatchId,SemId)
As
(
select id,StudentName,UniversityRegNo,AcademicBatch_Id,Semester_Id
from T_CFG_STUDENT_PERSONAL_DETAIL
where AcademicYear_Id=@pAcademicYear and Institution_Id=@pInsitutionId and Branch_Id=@pBranchId and YearOfStudy_Id=@pYearofStudyId 
)
,
regulationInfo(regId)
As
(
select Regulation_Id from T_CFG_ACADEMICBATCH d where d.Id=(select Top 1 StuBatchId from StuInfo)
)
,

CourseInfo(SubId,SubCode,SubName)
As
(
select c.Course_Id,c.Course_Code,c.Course_Name from T_COURSE_REGULATION_SYALLBUS_HDR a
join T_COURSE_REGULATION_SYALLBUS_DTL b on a.ID = b.ID
join T_CFG_COURSE_MASTER c on b.course_code=c.Course_Id
where a.Regulation_ID=(select TOP 1 regId from regulationInfo)  and
	  a.Institution_ID=@pInsitutionId and a.Branch_Id=@pBranchId and a.YearOfStudy_Id=@pYearofStudyId and a.Semester_ID=(select top 1 SemId from StuInfo) and c.Course_Code not in ('','lab','-','PR','P and','Library')

	  
)
,
pivotInfo ([StuTabId],[Registration_Number],StuName,[CourseName],Present,[Absent])
As
(

select StuTabId as [StuTabId], StuUnivRegNo as [Registration Number], StuName as [StudentName],SubName as [CourseName],isnull(P,0) as Present,isnull(Ab,0) as [Absent]
from
(
select stu.StuTabId, stu.StuUnivRegNo, stu.StuName,CourseInfo.SubName,atndnce.Attendance_Status,count(atndnce.Attendance_Status) as AttendanceStatus from T_ATTENDANCEENTRY atndnce 
join StuInfo stu on atndnce.StudentTable_Id= stu.StuTabId
join CourseInfo on atndnce.Course_Id=CourseInfo.SubId
where atndnce.attendance_date between CONVERT(varchar(50),@pFromDate,103) And CONVERT(varchar(50),@pToDate,103) group by stu.StuName,CourseInfo.SubName,atndnce.Attendance_Status,stu.StuUnivRegNo,stu.StuTabId -- order by atndnce.Course_Id,atndnce.StudentTable_Id
) as source
pivot (max(AttendanceStatus) for Attendance_Status in([P],[Ab])) as pvt

)
--select @cols1=[CourseName] from pivotInfo
--select  @cols1 = (STUFF((SELECT ',' + QUOTENAME(SubName) 
--	FROM CourseInfo order by SubName FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,''))

--SET @query1 = '
select [Registration_Number] as [Register Number],StuName as [Name of student], [Transforms and Partial Differential Equations],
[Environmental Science and Engineering],[Computer Architecture],[Programming and Data Structure II],[Database Management Systems],
[Analog and Digital Communication],[Programming and Data Structure Laboratory II],[Database Management Systems Laboratory]
from
(
select [StuTabId],[Registration_Number],StuName,CourseName,(convert(nvarchar(5),Attended) + '-' + convert(nvarchar(5),Conducted) +'-' +convert(nvarchar(10),[Attendance_Percentage])) as data
from
(select [StuTabId], [Registration_Number], StuName,CourseName,Present as Attended,[Absent] as Ab,
	   (Present+[Absent])as Conducted,
	   cast((Present*100)/((Present+[Absent])) as decimal(10,2))  as [Attendance_Percentage]
 from pivotInfo) as newDataTable
  --order by CourseName

) as newSource
 pivot(max(data) for CourseName in([Transforms and Partial Differential Equations],
[Environmental Science and Engineering],[Computer Architecture],[Programming and Data Structure II],[Database Management Systems],
[Analog and Digital Communication],[Programming and Data Structure Laboratory II],[Database Management Systems Laboratory])) as newPvt--'

 execute(@query1)
end

--exec dontDel 1,1,2,3,'2014-07-03','2014-07-23'
GO
/****** Object:  StoredProcedure [dbo].[getAbsentees]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
 CREATE procedure [dbo].[getAbsentees]
@pBranchId bigint,
@pYearOfStudyId bigint,
@pSemesterId bigint,
@pSectionId bigint,	
@pInstitutionId int

	
as
begin

Declare @programId bigint, @regulationId bigint;

SET @programId=(select Programme_Id from T_CFG_BRANCH where Branch_Id=@pBranchId and Status=1 and Delete_Flag=0);
SET @regulationId= (select CurrentRegulation from T_CFG_YEAROFSTUDY_MASTER where YearOfStudy_Id=@pYearOfStudyId and Programme_Id=@programId and Status=1 and Delete_Flag=0);

 With TGroupByMarks(Student_Details,Course_Name,Row_Num)
As
(
select Student_Details,Course_Name,ROW_NUMBER() over(order by Student_Details) Row_Num
 from ( select TSPD.StudentName+' '+TSPD.Initial as Student_Details,
 TCM.Course_Code +' - '+TCM.Course_ShortCode Course_Name
  from T_MARK_ENTRY TME
join T_CFG_STUDENT_PERSONAL_DETAIL TSPD on TME.StudentTable_Id = TSPD.Id
join T_CFG_COURSE_MASTER TCM on TME.Course_Id = TCM.Course_Id 
where TME.Is_Absent=1 and TME.Branch_Id=@pBranchId and TME.YearOfStudy_Id=@pYearOfStudyId and
	  TME.Semester_Id=@pSemesterId and TME.Section_Id=@pSectionId and TME.Institution_Id=@pInstitutionId and
	  TME.Exam_Id=(select TEM.Exam_Id from T_CFG_EXAM_MASTER TEM where TEM.Status=1 and TEM.Delete_Flag=0)
	  and TSPD.Status=1 and TSPD.Delete_Flag=0 and TSPD.TC_ISSUED=0
) temp
)
select * into finalTable from TGroupByMarks
DECLARE @cols AS NVARCHAR(MAX), @query AS NVARCHAR(MAX);

SET @cols = STUFF((SELECT distinct ',' + QUOTENAME(TCM.Course_Code +' - '+TCM.Course_ShortCode)FROM  T_COURSE_REGULATION_SYALLBUS_HDR TCRSH
join T_COURSE_REGULATION_SYALLBUS_DTL TCRSD on TCRSH.ID = TCRSD.ID
join T_CFG_COURSE_MASTER TCM on TCRSD.Course_Code = TCM.Course_Id 
where TCRSH.Institution_ID=@pInstitutionId and TCRSH.Branch_ID=@pBranchId and TCRSH.YearOfStudy_ID=@pYearOfStudyId and TCRSH.Regulation_ID=@regulationId
 and TCRSH.Semester_ID=@pSemesterId and TCRSH.status=1 and TCRSH.Flag=0 and TCRSD.FLAG=0 and TCRSD.STATUS=1
 and TCM.Course_ShortCode not in('Lab','LIB','NPTEL','PR')  FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'')  

 SET @query='
select '+@cols+'
 from (select  Student_Details,Course_Name,Row_Num from finalTable) c 
 pivot 
 (
 max(Student_Details) for Course_Name in('+@cols+')
 ) as pivotTable'

  execute(@query)
   drop table finalTable

   end


   --exec getAbsentees 1,4,7,1,1


   --select * from T_COURSE_REGULATION_SYALLBUS_HDR

   --select * from T_CFG_COURSE_MASTER





GO
/****** Object:  StoredProcedure [dbo].[getAttendanceIndividual]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE proc [dbo].[getAttendanceIndividual]
@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),
	@pSectionId NVARCHAR(50),
	@ExamFdate nvarchar(50),
	@ExamTdate nvarchar(50),
	@PercentageRange nvarchar(10),
	@pOrderById nvarchar(50)
	

	
as
begin
DECLARE @cols AS NVARCHAR(MAX), @query AS NVARCHAR(MAX),@pOrderByName AS NVARCHAR(50),@pAppText NVARCHAR(MAX); 
SET @pOrderByName=(select ORDER_BY_VALUE from T_CFG_ORDER_BY_DETAILS where ID=@pOrderById)

if(@pOrderByName='UniversityRegNo')
begin

SET @pAppText=' RegNo'

end
else if(@pOrderByName='StudentId_Rollno')
begin
SET @pAppText=' StudentId_Rollno';
end
else if(@pOrderByName='StudentName')
begin
SET @pAppText=' StudentName';
end
else if(@pOrderByName='SNo')
begin
SET @pAppText=' SNO';
end
else if(@pOrderByName='Male')
begin
SET @pAppText=' [Gender] DESC';
end
else if(@pOrderByName='Female')
begin
SET @pAppText=' [Gender]';
end

if(@pSectionId !='0')
begin




SET @cols = STUFF((SELECT ',' + QUOTENAME(CM.Course_Code + '-'+CM.Course_ShortCode) FROM  T_CFG_COURSE_MASTER CM
join T_ATTENDANCEENTRY ME (NOLOCK) on CM.Course_Id=ME.Course_Id 
where ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId and ME.Section_Id=@pSectionId 
and  CM.Is_Exam=1 
--CM.Course_Code not in ('','lab','-','PR','P and','Library') 
group by CM.Course_ShortCode,CM.Course_Code
order by MAX(CM.TutorialType) desc,CM.Course_ShortCode
FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'')  
 
 SET @query='
SELECT distinct SNO,StudentId_Rollno,Gender,Section_Id,RegNo,StudentId,StudentName,[Section],'+@cols+' 
 FROM (SELECT SNO,StudentName,StudentId_Rollno,Gender,Section_Id,[Section],StudentId,RegNo,PercentageOfAttendance,coursename FROM  fncAttendance('+@pBranchId+','+@pYearOfStudyId+','+@pSemesterId+','+@pSectionId+','''+@ExamFdate+''','''+@ExamTdate+''')) as a
 PIVOT
 (max(PercentageOfAttendance) FOR  coursename IN ('+@cols+')) as pvt
 
ORDER BY Section_Id,'+@pAppText


execute(@query)

----To get OverAll attendance
select distinct sm.StudentName+' '+sm.Initial as StudentName,sm.Id,sm.UniversityRegNo as RegNo,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)as presentPeriod,
--sum(case when ae.Attendance_Status='AB' then 1 else 0 end) as absentPeriod,
sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as totalPeriod,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as Percentage

from T_ATTENDANCEENTRY AE (NOLOCK)
join T_CFG_COURSE_MASTER CM on AE.Course_Id=CM.Course_Id
join T_CFG_STUDENT_PERSONAL_DETAIL SM on AE.StudentTable_Id = SM.Id
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId and AE.Section_Id=@pSectionId and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103) 
group by sm.StudentName,sm.Initial,sm.Id,sm.UniversityRegNo
--having(sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)<@PercentageRange)



--90 % to 85 %
select distinct 'Less than - 90% upto 85%'  as [Desc],
sum(case when AE.Attendance_Status='P' then 1 else 0 end)as presentPeriod,sm.StudentName+' '+sm.Initial as StudentName,SECM.Section_Name as Section,sm.Id,
--sum(case when ae.Attendance_Status='AB' then 1 else 0 end) as absentPeriod,
sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as totalPeriod,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as Percentage

from T_ATTENDANCEENTRY AE (NOLOCK)
join T_CFG_COURSE_MASTER CM on AE.Course_Id=CM.Course_Id
join T_CFG_STUDENT_PERSONAL_DETAIL SM on AE.StudentTable_Id = SM.Id
join T_CFG_SECTION_MASTER SECM on SM.Section_Id=SECM.ID
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId and AE.Section_Id=@pSectionId and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103) 
and SM.Status=1 and SM.Delete_Flag=0 and SM.TC_ISSUED=0
group by sm.StudentName,sm.Initial,sm.Id,SECM.Section_Name
having(sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)>=85 and
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)<=90)

-----------------3.>85 to <=80
select distinct 'Less than - 85% upto 80%'  as [Desc],
sum(case when AE.Attendance_Status='P' then 1 else 0 end)as presentPeriod, sm.StudentName+' '+sm.Initial as StudentName,SECM.Section_Name as Section,sm.Id,
--sum(case when ae.Attendance_Status='AB' then 1 else 0 end) as absentPeriod,
sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as totalPeriod,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as Percentage

from T_ATTENDANCEENTRY AE (NOLOCK)
join T_CFG_COURSE_MASTER CM on AE.Course_Id=CM.Course_Id
join T_CFG_STUDENT_PERSONAL_DETAIL SM on AE.StudentTable_Id = SM.Id
join T_CFG_SECTION_MASTER SECM on SM.Section_Id=SECM.ID
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId and AE.Section_Id=@pSectionId and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103) 
and SM.Status=1 and SM.Delete_Flag=0 and SM.TC_ISSUED=0
group by sm.StudentName,sm.Initial,sm.Id,SECM.Section_Name
having(sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)>80 and
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)<=85)

----------------4.>90
select distinct 'Less than - 80% upto 75%'  as [Desc],
sum(case when AE.Attendance_Status='P' then 1 else 0 end)as presentPeriod, sm.StudentName+' '+sm.Initial as StudentName,SECM.Section_Name as Section,sm.Id,
--sum(case when ae.Attendance_Status='AB' then 1 else 0 end) as absentPeriod,
sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as totalPeriod,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as Percentage

from T_ATTENDANCEENTRY AE (NOLOCK)
join T_CFG_COURSE_MASTER CM on AE.Course_Id=CM.Course_Id
join T_CFG_STUDENT_PERSONAL_DETAIL SM on AE.StudentTable_Id = SM.Id
join T_CFG_SECTION_MASTER SECM on SM.Section_Id=SECM.ID
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId and AE.Section_Id=@pSectionId and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103) 
and SM.Status=1 and SM.Delete_Flag=0 and SM.TC_ISSUED=0
group by sm.StudentName,sm.Initial,sm.Id,SECM.Section_Name
having(sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)>75 and
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)<=80)

---Below 75
select distinct 'Below 75%' as [Desc],
sum(case when AE.Attendance_Status='P' then 1 else 0 end)as presentPeriod,sm.StudentName+' '+sm.Initial as StudentName,SECM.Section_Name as Section,sm.Id,
--sum(case when ae.Attendance_Status='AB' then 1 else 0 end) as absentPeriod,
sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as totalPeriod,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as Percentage

from T_ATTENDANCEENTRY AE (NOLOCK)
join T_CFG_COURSE_MASTER CM on AE.Course_Id=CM.Course_Id
join T_CFG_STUDENT_PERSONAL_DETAIL SM on AE.StudentTable_Id = SM.Id
join T_CFG_SECTION_MASTER SECM on SM.Section_Id=SECM.ID
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId and AE.Section_Id=@pSectionId and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103) 
and SM.Status=1 and SM.Delete_Flag=0 and SM.TC_ISSUED=0
group by sm.StudentName,sm.Initial,sm.Id,SECM.Section_Name
having(sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)<convert(int,@PercentageRange))

------To get no.of.Working Days

select count(temp.workingDates) as numberOfWorkingDays from(select ae.Attendance_Date, count(ae.Attendance_Date) as workingDates
 from T_ATTENDANCEENTRY AE (NOLOCK)
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId and AE.Section_Id=@pSectionId and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103)
group by ae.Attendance_Date) as temp


--attendance count
select distinct studentTable_id,count(numberOfWorkingDays)as noOfPresentDays from [fncAttendanceAvg](@pBranchId,@pYearOfStudyId,@pSemesterId,@pSectionId,@ExamFdate,@ExamTdate) group by studentTable_id



 end

 else
 begin

 --SET @cols = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_ShortCode) FROM  T_CFG_COURSE_MASTER CM
--join T_ATTENDANCEENTRY ME on CM.Course_Id=ME.Course_Id 
--where ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId and ME.Section_Id=@pSectionId 
--FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'')

SET @cols = STUFF((SELECT  ',' + QUOTENAME(CM.Course_Code + '-'+CM.Course_ShortCode) FROM  T_CFG_COURSE_MASTER CM
join T_ATTENDANCEENTRY ME (NOLOCK) on CM.Course_Id=ME.Course_Id 
where ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId 
and  CM.Is_Exam=1 
--CM.Course_Code not in ('','lab','-','PR','P and','Library')
group by CM.Course_ShortCode,CM.Course_Code
order by  MAX(CM.TutorialType) desc,CM.Course_ShortCode
FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'')  


 
 SET @query='
SELECT distinct SNO,StudentId_Rollno,Gender,Section_Id,RegNo,StudentId,StudentName,[Section],'+@cols+' 
 FROM (SELECT SNO,StudentName,StudentId_Rollno,Gender,Section_Id,[Section],StudentId,RegNo,PercentageOfAttendance,coursename FROM  fncAttendanceAll('+@pBranchId+','+@pYearOfStudyId+','+@pSemesterId+','+@pSectionId+','''+@ExamFdate+''','''+@ExamTdate+''')) as a
 PIVOT
 (max(PercentageOfAttendance) FOR  coursename IN ('+@cols+')) as pvt
 
ORDER BY Section_Id,'+@pAppText
--select @query
--print(@query)
execute(@query)

----To get OverAll attendance
select distinct sm.StudentName+' '+sm.Initial as StudentName,sm.Id,sm.UniversityRegNo as RegNo,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)as presentPeriod,
--sum(case when ae.Attendance_Status='AB' then 1 else 0 end) as absentPeriod,
sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as totalPeriod,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as Percentage

from T_ATTENDANCEENTRY AE (NOLOCK)
join T_CFG_COURSE_MASTER CM on AE.Course_Id=CM.Course_Id
join T_CFG_STUDENT_PERSONAL_DETAIL SM on AE.StudentTable_Id = SM.Id
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103) 
group by sm.StudentName,sm.Initial,sm.Id,sm.UniversityRegNo
--having(sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)<@PercentageRange)



--90 % to 85 %
select distinct 'Less than - 90% upto 85%'  as [Desc],
sum(case when AE.Attendance_Status='P' then 1 else 0 end)as presentPeriod,sm.StudentName+' '+sm.Initial as StudentName,SECM.Section_Name as Section,sm.Id,
--sum(case when ae.Attendance_Status='AB' then 1 else 0 end) as absentPeriod,
sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as totalPeriod,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as Percentage

from T_ATTENDANCEENTRY AE (NOLOCK)
join T_CFG_COURSE_MASTER CM on AE.Course_Id=CM.Course_Id
join T_CFG_STUDENT_PERSONAL_DETAIL SM on AE.StudentTable_Id = SM.Id
join T_CFG_SECTION_MASTER SECM on SM.Section_Id=SECM.ID
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId  and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103) 
and SM.Status=1 and SM.Delete_Flag=0 and SM.TC_ISSUED=0
group by sm.StudentName,sm.Initial,sm.Id,SECM.Section_Name
having(sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)>=85 and
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)<=90)

-----------------3.>85 to <=80
select distinct 'Less than - 85% upto 80%'  as [Desc],
sum(case when AE.Attendance_Status='P' then 1 else 0 end)as presentPeriod, sm.StudentName+' '+sm.Initial as StudentName,SECM.Section_Name as Section,sm.Id,
--sum(case when ae.Attendance_Status='AB' then 1 else 0 end) as absentPeriod,
sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as totalPeriod,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as Percentage

from T_ATTENDANCEENTRY AE (NOLOCK)
join T_CFG_COURSE_MASTER CM on AE.Course_Id=CM.Course_Id
join T_CFG_STUDENT_PERSONAL_DETAIL SM on AE.StudentTable_Id = SM.Id
join T_CFG_SECTION_MASTER SECM on SM.Section_Id=SECM.ID
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103) 
and SM.Status=1 and SM.Delete_Flag=0 and SM.TC_ISSUED=0
group by sm.StudentName,sm.Initial,sm.Id,SECM.Section_Name
having(sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)>80 and
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)<=85)

----------------4.>90
select distinct 'Less than - 80% upto 75%'  as [Desc],
sum(case when AE.Attendance_Status='P' then 1 else 0 end)as presentPeriod, sm.StudentName+' '+sm.Initial as StudentName,SECM.Section_Name as Section,sm.Id,
--sum(case when ae.Attendance_Status='AB' then 1 else 0 end) as absentPeriod,
sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as totalPeriod,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as Percentage

from T_ATTENDANCEENTRY AE (NOLOCK)
join T_CFG_COURSE_MASTER CM on AE.Course_Id=CM.Course_Id
join T_CFG_STUDENT_PERSONAL_DETAIL SM on AE.StudentTable_Id = SM.Id
join T_CFG_SECTION_MASTER SECM on SM.Section_Id=SECM.ID
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId  and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103) 
and SM.Status=1 and SM.Delete_Flag=0 and SM.TC_ISSUED=0
group by sm.StudentName,sm.Initial,sm.Id,SECM.Section_Name
having(sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)>75 and
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)<=80)

---Below 75
select distinct 'Below 75%' as [Desc],
sum(case when AE.Attendance_Status='P' then 1 else 0 end)as presentPeriod,sm.StudentName+' '+sm.Initial as StudentName,SECM.Section_Name as Section,sm.Id,
--sum(case when ae.Attendance_Status='AB' then 1 else 0 end) as absentPeriod,
sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as totalPeriod,
sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end) as Percentage

from T_ATTENDANCEENTRY AE (NOLOCK)
join T_CFG_COURSE_MASTER CM on AE.Course_Id=CM.Course_Id
join T_CFG_STUDENT_PERSONAL_DETAIL SM on AE.StudentTable_Id = SM.Id
join T_CFG_SECTION_MASTER SECM on SM.Section_Id=SECM.ID
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId  and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103) 
and SM.Status=1 and SM.Delete_Flag=0 and SM.TC_ISSUED=0
group by sm.StudentName,sm.Initial,sm.Id,SECM.Section_Name
having(sum(case when AE.Attendance_Status='P' then 1 else 0 end)*100/sum( case when AE.Attendance_Status='P' then 1 else 0 end + case when ae.Attendance_Status='AB' then 1 else 0 end)<convert(int,@PercentageRange))

------To get no.of.Working Days

select count(temp.workingDates) as numberOfWorkingDays from(select ae.Attendance_Date, count(ae.Attendance_Date) as workingDates  
from T_ATTENDANCEENTRY AE (NOLOCK)
where AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId  and CONVERT(date,AE.Attendance_Date,103) between CONVERT(varchar(50),@ExamFdate,103)  and CONVERT(varchar(50),@ExamTdate,103)
group by ae.Attendance_Date) as temp


--attendance count
select distinct studentTable_id,count(numberOfWorkingDays)as noOfPresentDays from [fncAttendanceAvgAll](@pBranchId,@pYearOfStudyId,@pSemesterId,@pSectionId,@ExamFdate,@ExamTdate) group by studentTable_id

 end
 end

 --exec getAttendanceIndividual 1,4,7,0,'2014-08-04','2014-09-15',60,4
 --select * from fncAttendance(1,2,3,3,'2014-03-10','2014-09-10')

 




--select  Course_ShortCode,TutorialType from T_CFG_COURSE_MASTER 
--group by Course_ShortCode,TutorialType
--order by MAX(TutorialType) desc 


--select  Course_ShortCode,TutorialType from T_CFG_COURSE_MASTER 
--group by Course_ShortCode,TutorialType
--order by MIN(TutorialType) desc 
GO
/****** Object:  StoredProcedure [dbo].[GetAuAttendanceMonthlyReport]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE Procedure [dbo].[GetAuAttendanceMonthlyReport]

@pInsitutionId nvarchar(5), @pBranchId nvarchar(5), @pYearofStudyId nvarchar(5),@pSemester nvarchar(5),@pSectionId nvarchar(5), @pAcademicYear nvarchar(5),@pFromDate nvarchar(20),
@pToDate nvarchar(20),@pOrderById nvarchar(5)
As Begin
Declare @pRegulation nvarchar(5),@pOrderByName AS NVARCHAR(50),@pAppText NVARCHAR(MAX),@testt nvarchar(MAX);

SET @pOrderByName=(select ORDER_BY_VALUE from T_CFG_ORDER_BY_DETAILS where ID=@pOrderById)

SET @pRegulation=(select Regulation_Id from T_CFG_ACADEMICBATCH d
				 where d.Id=(select Top 1 AcademicBatch_Id from T_CFG_STUDENT_PERSONAL_DETAIL
						where AcademicYear_Id=@pAcademicYear and Institution_Id=@pInsitutionId and Branch_Id=@pBranchId and YearOfStudy_Id=@pYearofStudyId
						and Status=1 and Delete_Flag=0 and TC_ISSUED=0));
--SET @pSemester=(select Top 1 Semester_Id from T_CFG_STUDENT_PERSONAL_DETAIL
--						where AcademicYear_Id=@pAcademicYear and Institution_Id=@pInsitutionId and Branch_Id=@pBranchId and YearOfStudy_Id=@pYearofStudyId
--						and Status=1 and Delete_Flag=0 and TC_ISSUED=0);

if(@pOrderByName='UniversityRegNo')
begin

SET @pAppText=',[Registration_Number]';

end
else if(@pOrderByName='StudentId_Rollno')
begin
SET @pAppText=',[Student Id]';
end
else if(@pOrderByName='StudentName')
begin
SET @pAppText=',[StudentName]';
end
else if(@pOrderByName='SNo')
begin
SET @pAppText=',SNo';
end
else if(@pOrderByName='Male')
begin
SET @pAppText=',[Gender] DESC';
end
else if(@pOrderByName='Female')
begin
SET @pAppText=',[Gender]';
end

if(@pSectionId=0)
begin 
SET @testt='

With StuInfo([Stu_Tab_Id],Gender,SNo,[Student Id],[Registration_Number],[Section],[StudentName])
As
(
select stu.Id [Stu_Tab_Id],stu.Gender,stu.SNo,stu.StudentId_Rollno [Student Id],stu.UniversityRegNo [Registration_Number],s.Section_Name Section,stu.StudentName+'' ''+stu.Initial as [StudentName]
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_CFG_SECTION_MASTER S on stu.Section_Id=s.ID
where stu.AcademicYear_Id='+@pAcademicYear+' and stu.Institution_Id='+@pInsitutionId+' and stu.Branch_Id='+@pBranchId+' and stu.YearOfStudy_Id='+@pYearofStudyId+'  and  stu.Semester_ID='+@pSemester+' 
and stu.Status=1 and stu.Delete_Flag=0 and stu.TC_ISSUED=0 
),
CourseInfo(CourseId,[CourseName])
As
(
select DTL.Course_Code CourseId,c.Course_Code +'' | ''+ c.Course_Name [CourseName] from  T_COURSE_REGULATION_SYALLBUS_HDR HDR
join T_COURSE_REGULATION_SYALLBUS_DTL DTL on DTL.ID = HDR.ID
join T_CFG_COURSE_MASTER c on DTL.course_code=c.Course_Id
where  HDR.Regulation_ID='+@pRegulation+' and HDR.Institution_ID='+@pInsitutionId+' and HDR.Branch_Id='+@pBranchId+' and HDR.YearOfStudy_Id='+@pYearofStudyId+' and HDR.Semester_ID='+@pSemester+'
and  c.Is_Exam=1
),
AttenInfo([Stu_Tab_Id],Gender,SNo,[Student Id],[Registration_Number],[Section],[StudentName],CourseId, [CourseName],Attended,Conducted)
As
(
select  StuInfo.*,CourseInfo.*, sum(case when att.Attendance_Status=''P'' then 1 else 0 end) Attended,count(att.Attendance_Status) Conducted
from T_ATTENDANCEENTRY att (NOLOCK)
join StuInfo on att.StudentTable_Id=StuInfo.Stu_Tab_Id
join CourseInfo on att.Course_Id=CourseInfo.CourseId
where att.attendance_date between  CONVERT(varchar(50),'''+@pFromDate+''',103) And CONVERT(varchar(50),'''+@pToDate+''',103)
group by att.StudentTable_Id,att.Course_Id,
StuInfo.[Stu_Tab_Id],StuInfo.Gender,StuInfo.SNo,StuInfo.[Student Id],StuInfo.[Registration_Number],StuInfo.[Section],StuInfo.[StudentName],
CourseInfo.CourseId,CourseInfo.[CourseName]
)
select *,cast((cast((Attended*100) as decimal(10,2))/cast(Conducted as decimal(10,2)))as decimal(10,2))  as [Attendance_Percentage] 
from AttenInfo order by [CourseName],[Section] ' +@pAppText

  print @testt
 exec(@testt)
end
else
begin
SET @testt='

With StuInfo([Stu_Tab_Id],Gender,SNo,[Student Id],[Registration_Number],[Section],[StudentName])
As
(
select stu.Id [Stu_Tab_Id],stu.Gender,stu.SNo,stu.StudentId_Rollno [Student Id],stu.UniversityRegNo [Registration_Number],s.Section_Name Section,stu.StudentName+'' ''+stu.Initial as [StudentName]
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_CFG_SECTION_MASTER S on stu.Section_Id=s.ID
where stu.AcademicYear_Id='+@pAcademicYear+' and stu.Institution_Id='+@pInsitutionId+' and stu.Branch_Id='+@pBranchId+' and stu.YearOfStudy_Id='+@pYearofStudyId+'  and  stu.Semester_ID='+@pSemester+' and stu.Section_Id='+@pSectionId+'
and stu.Status=1 and stu.Delete_Flag=0 and stu.TC_ISSUED=0 
),
CourseInfo(CourseId,[CourseName])
As
(
select DTL.Course_Code CourseId,c.Course_Code +'' | ''+ c.Course_Name [CourseName] from  T_COURSE_REGULATION_SYALLBUS_HDR HDR
join T_COURSE_REGULATION_SYALLBUS_DTL DTL on DTL.ID = HDR.ID
join T_CFG_COURSE_MASTER c on DTL.course_code=c.Course_Id
where  HDR.Regulation_ID='+@pRegulation+' and HDR.Institution_ID='+@pInsitutionId+' and HDR.Branch_Id='+@pBranchId+' and HDR.YearOfStudy_Id='+@pYearofStudyId+' and HDR.Semester_ID='+@pSemester+'
and  c.Is_Exam=1
),
AttenInfo([Stu_Tab_Id],Gender,SNo,[Student Id],[Registration_Number],[Section],[StudentName],CourseId, [CourseName],Attended,Conducted)
As
(
select  StuInfo.*,CourseInfo.*, sum(case when att.Attendance_Status=''P'' then 1 else 0 end) Attended,count(att.Attendance_Status) Conducted
from T_ATTENDANCEENTRY att (NOLOCK)
join StuInfo on att.StudentTable_Id=StuInfo.Stu_Tab_Id
join CourseInfo on att.Course_Id=CourseInfo.CourseId
where att.attendance_date between  CONVERT(varchar(50),'''+@pFromDate+''',103) And CONVERT(varchar(50),'''+@pToDate+''',103)
group by att.StudentTable_Id,att.Course_Id,
StuInfo.[Stu_Tab_Id],StuInfo.Gender,StuInfo.SNo,StuInfo.[Student Id],StuInfo.[Registration_Number],StuInfo.[Section],StuInfo.[StudentName],
CourseInfo.CourseId,CourseInfo.[CourseName]
)
select *,cast((cast((Attended*100) as decimal(10,2))/cast(Conducted as decimal(10,2)))as decimal(10,2))  as [Attendance_Percentage] 
from AttenInfo order by [CourseName] ' +@pAppText

 print @testt
  exec(@testt)
end
 





end






--exec GetAuAttendanceMonthlyReport 1,1,2,4,1,3,'2015-01-01','2015-05-29',3









GO
/****** Object:  StoredProcedure [dbo].[GetBindMenu]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[GetBindMenu]
	-- Add the parameters for the stored procedure here
	@pRoleId varchar(50),
	@pUserId bigint
AS
BEGIN
	SET NOCOUNT ON;   
	select  M.id as 'ModuleId',SM.id as 'SubModuleId',s.Id as 'ScreenId',s.OrderBy as 'ScreenOrder', M.ModuleName as 'ModuleName',SM.SubModuleName as 'SubModuleName',S.ScreenName as 'ScreenName',S.action as 'action',
	S.Controller as 'Controller',M.ModuleIcon as 'ModuleIcon',SM.SubModuleIcon as 'SubModuleIcon',S.ScreenIcon as 'ScreenIcon',
	--RP.PrivilegeId as 'PrivilegeId',
	--RP.bCreate as 'Create',RP.bView as 'View',RP.bUpdate as 'Update',RP.bDelete as 'Delete',
	M.OrderBy as 'ModuleOrderBy',SM.OrderBy as 'SubModuleOrderBy'
    from T_CFG_ROLESPERMISSION RP,T_CFG_MODULE M,T_CFG_SUBMODULE SM,T_CFG_SCREEN S
    where RP.ModuleId=M.id and RP.SubModuleId=SM.id and RP.ScreenId=S.id and S.IsDeleted='False' and RP.IsDeleted='False'  and RP.bCreate='True' and RP.RoleId in(Select s from SplitString(@pRoleId,','))
    
    union
    
    select  UM.id as 'ModuleId',USM.id as 'SubModuleId',Us.Id as 'ScreenId',US.OrderBy as 'ScreenOrder', UM.ModuleName as 'ModuleName',USM.SubModuleName as 'SubModuleName',US.ScreenName as 'ScreenName',US.action as 'action',
    US.Controller as 'Controller',UM.ModuleIcon as 'ModuleIcon',USM.SubModuleIcon as 'SubModuleIcon',US.ScreenIcon as 'ScreenIcon',
    --UP.PrivilegeId  as 'PrivilegeId',
    --UP.bCreate as 'Create',UP.bView as 'View',UP.bUpdate as 'Update',UP.bDelete as 'Delete', 
    UM.OrderBy as 'ModuleOrderBy',USM.OrderBy as 'SubModuleOrderBy'
    from T_CFG_USERPERMISSION UP,T_CFG_MODULE UM,T_CFG_SUBMODULE USM,T_CFG_SCREEN US 
    where UP.ModuleId=UM.id and UP.SubModuleId=USM.id and UP.ScreenId=US.id and UP.UserId=@pUserId order by ModuleOrderBy asc, SubModuleOrderBy asc,ScreenOrder asc
     --in(Select s from SplitString(@pRoleId,','))  
	
END


--exec GetBindMenu 1,2



GO
/****** Object:  StoredProcedure [dbo].[GetBranchWiseToppersList]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE procedure [dbo].[GetBranchWiseToppersList]
--@pInsitutionId bigint, @pBranchId bigint, @pYearofStudyId bigint,@pSemester bigint
As Begin
Declare @pRegulation bigint,@pAcademicYear bigint;
DECLARE @cols1 AS NVARCHAR(MAX), @query1 AS NVARCHAR(MAX); 
SET @pAcademicYear=(select ID from T_CFG_ACADEMIC_YEAR Where Academic_year_Status = 1);

SET @pRegulation=(select Regulation_Id from T_CFG_ACADEMICBATCH d
				 where d.Id=(select Top 1 AcademicBatch_Id from T_CFG_STUDENT_PERSONAL_DETAIL
						where AcademicYear_Id=@pAcademicYear and Institution_Id=1 and Branch_Id=4 and YearOfStudy_Id=2 and Status=1 and Delete_Flag=0 and TC_ISSUED=0));
SET @cols1 = STUFF((SELECT distinct ','+ QUOTENAME( c.Course_Name) from T_COURSE_REGULATION_SYALLBUS_HDR a
join T_COURSE_REGULATION_SYALLBUS_DTL b on  a.ID=b.ID
join T_CFG_COURSE_MASTER c on b.Course_Code = c.Course_Id
where a.Regulation_ID=6 and a.Institution_ID=1 and a.Branch_Id=4 and a.YearOfStudy_Id=2 and a.Semester_Id=3 and c.Course_Code not in ('','lab','-','PR','P and','Library')
FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 
 SET @query1 = '
select UniversityRegNo,StudentName,Section_Name,'+@cols1+' 
from
(
select UniversityRegNo,StudentName,Section_Name,Course_Name,Marks,[rank_No]
 from
 (
 select stu.UniversityRegNo,stu.StudentName+'' ''+stu.Initial as StudentName,sec.Section_Name,cou.Course_Name,mark.Marks,
DENSE_RANK() over (partition by mark.course_id,stu.section_id order by mark.marks desc) as [rank_No] 
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_MARK_ENTRY mark on stu.Id = mark.StudentTable_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
join T_CFG_COURSE_MASTER cou on mark.Course_Id=cou.Course_Id
where stu.Branch_Id=6 and stu.YearOfStudy_Id=2 and stu.Semester_Id=3 and mark.Exam_Id=1 
and stu.Status=1 and stu.Delete_Flag=0 and stu.TC_ISSUED=0
) as source where [rank_No]<4
) as newSource
pivot(max(marks) for Course_Name in ('+@cols1+' )) as pvt'

PRINT @query1

execute(@query1)
end
GO
/****** Object:  StoredProcedure [dbo].[GetDailyReport]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[GetDailyReport]
@Branch varchar(50),
@year varchar(50),
@semester varchar(50),
@section varchar(50),
@Attendancedate varchar(50)

AS
Begin

SELECT DISTINCT 
                         a.StudentTable_Id, b.OD_Remarks, b.OD_Periods, dbo.T_CFG_STUDENT_PERSONAL_DETAIL.UniversityRegNo, 
                         dbo.T_CFG_STUDENT_PERSONAL_DETAIL.StudentName+' '+dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Initial as StudentName, dbo.T_CFG_STUDENT_PERSONAL_DETAIL.MobileNo, a.Branch_Id, a.YearOfStudy_Id, 
                         a.Semester_Id, a.Section_Id
FROM            dbo.T_ATTENDANCEENTRY AS a INNER JOIN
                         dbo.T_OD_DETAILS AS b ON a.Attendance_Date = b.From_Date AND a.StudentTable_Id = b.Student_Id AND a.Attendance_Status = 'AB' INNER JOIN
                         dbo.T_CFG_STUDENT_PERSONAL_DETAIL ON a.StudentTable_Id = dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Id
WHERE        (a.Attendance_Date = @Attendancedate)  and  a.Branch_Id=@Branch and a.YearOfStudy_Id=@year and 
                         a.Semester_Id=@semester and a.Section_Id=@section 
						 and dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Status=1 and
						 dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Delete_Flag=0 and dbo.T_CFG_STUDENT_PERSONAL_DETAIL.TC_ISSUED=0


End

--exec GetDailyReport 8,2,3,3,'2014-08-25'

GO
/****** Object:  StoredProcedure [dbo].[getExamAbsentees]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[getExamAbsentees]
@pBranchId nvarchar(50),
@pYearOfStudyId nvarchar(50),
@pSemesterId nvarchar(50),
@pSectionId nvarchar(50),	
@pInstitutionId nvarchar(50),
@pExamId nvarchar(50)
as
begin

DECLARE @V_ExamId nvarchar(50),@V_AcademicYearId nvarchar(50);

--SET @V_ExamId=(select TEM.Exam_Id from T_CFG_EXAM_MASTER TEM where TEM.Status=1 and TEM.Delete_Flag=0);
SET @V_ExamId=@pExamId;

SET @V_AcademicYearId=(select ID from T_CFG_ACADEMIC_YEAR WHERE Status=1 AND flag=0 AND Academic_Year_Status=1)

DECLARE @cols AS NVARCHAR(MAX), @query AS NVARCHAR(MAX);

Declare @programId bigint, @regulationId bigint;

SET @programId=(select Programme_Id from T_CFG_BRANCH where Branch_Id=@pBranchId and Status=1 and Delete_Flag=0);
SET @regulationId= (select CurrentRegulation from T_CFG_YEAROFSTUDY_MASTER where YearOfStudy_Id=@pYearOfStudyId and Programme_Id=@programId and Status=1 and Delete_Flag=0);
SET @cols = STUFF((SELECT distinct ',' + QUOTENAME (TCM.Course_Code +' - '+TCM.Course_ShortCode) FROM  T_COURSE_REGULATION_SYALLBUS_HDR TCRSH
join T_COURSE_REGULATION_SYALLBUS_DTL TCRSD on TCRSH.ID = TCRSD.ID
join T_CFG_COURSE_MASTER TCM on TCRSD.Course_Code = TCM.Course_Id 
where TCRSH.Institution_ID=@pInstitutionId and TCRSH.Branch_ID=@pBranchId and TCRSH.YearOfStudy_ID=@pYearOfStudyId and TCRSH.Regulation_ID=@regulationId
 and TCRSH.Semester_ID=@pSemesterId and TCRSH.status=1 and TCRSH.Flag=0 and TCRSD.FLAG=0 and TCRSD.STATUS=1 and TCM.TutorialType!='Practical'
  and TCM.Is_Exam=1  FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'')

if(@pSectionId!= '0')
begin
SET @query='
SELECT   '+ISNULL(@cols,'')+'
FROM
(

SELECT  TME.Is_Absent as AB, TSPD.StudentId_Rollno, ISNULL((TSPD.StudentId_Rollno + ''-'' + TSPD.StudentName+'' ''+ TSPD.Initial ),'''') AS STUINFO , ISNULL((TCM.Course_Code + '' - '' + TCM.Course_ShortCode),'''') AS COURSEINFO  FROM T_MARK_ENTRY TME
JOIN T_CFG_STUDENT_PERSONAL_DETAIL TSPD ON TME.StudentTable_Id=TSPD.Id
JOIN T_CFG_COURSE_MASTER TCM ON TME.Course_Id=TCM.Course_Id
WHERE TME.Is_Absent=1 AND TME.Institution_Id='+@pInstitutionId+' AND TME.AcademicYear_Id='+@V_AcademicYearId+' AND TME.Branch_Id='+@pBranchId+' AND TME.YearOfStudy_Id='+@pYearOfStudyId+'
 AND TME.Semester_Id='+@pSemesterId+'
AND TSPD.Section_Id='+@pSectionId+' AND TME.Exam_Id='+@V_ExamId+'  and TCM.TutorialType=''Theoretical''


) T
pivot(max([STUINFO]) for COURSEINFO in( '+@cols+')) as Dynamicpivot'
end
else
begin


SET @query='
SELECT   '+ISNULL(@cols,'')+'
FROM
(

SELECT  TME.Is_Absent as AB, TSPD.StudentId_Rollno, ISNULL((TSPD.StudentId_Rollno + ''-'' + TSPD.StudentName+'' ''+ TSPD.Initial ),'''') AS STUINFO , ISNULL((TCM.Course_Code + '' - '' + TCM.Course_ShortCode),'''') AS COURSEINFO  FROM T_MARK_ENTRY TME
JOIN T_CFG_STUDENT_PERSONAL_DETAIL TSPD ON TME.StudentTable_Id=TSPD.Id
JOIN T_CFG_COURSE_MASTER TCM ON TME.Course_Id=TCM.Course_Id
WHERE TME.Is_Absent=1 AND TME.Institution_Id='+@pInstitutionId+' AND TME.AcademicYear_Id='+@V_AcademicYearId+' AND TME.Branch_Id='+@pBranchId+' AND TME.YearOfStudy_Id='+@pYearOfStudyId+'
 AND TME.Semester_Id='+@pSemesterId+'
AND TME.Exam_Id='+@V_ExamId+' and TCM.TutorialType!=''Practical''

) T
pivot(max([STUINFO]) for COURSEINFO in( '+@cols+')) as Dynamicpivot'
end
PRINT @query
 execute(@query)

 end

-- exec getExamAbsentees 8,4,7,1,1,1



--select * from T_CFG_COURSE_MASTER where Course_ShortCode='DC LAB' Studentid_Rollno like '%NAIT2B33%'

GO
/****** Object:  StoredProcedure [dbo].[GetFailStudents]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE Procedure [dbo].[GetFailStudents]
@pProgmId nvarchar(10),
@pYearId NVARCHAR(10),
@PSemesterId NVARCHAR(10)
,@pExamId NVARCHAR(10)

As Begin

DECLARE @cols1 AS NVARCHAR(MAX), @query1 AS NVARCHAR(MAX); 
SET @cols1 = STUFF((SELECT ',' + QUOTENAME(tBranch.Branch_Code) FROM
	 T_CFG_BRANCH tBranch 
	where tBranch.Programme_Id =@pProgmId and  tBranch.Status=1 and tBranch.Delete_Flag=0   order by tBranch.Branch_Name FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'')

	--select @cols1
SET @query1 = 'select  '+@cols1+'
from
(
SELECT ([StudentName]+'' - ''+convert(nvarchar(10),[FailCount])+'' - ''+Course+'' - ''+[StudentId_Rollno] +'' - ''+[YearOfStudy_Name] +'' - ''+[Section_Name]) As [UniversityRegNo]       
      ,*
  FROM fGetFailStudents('+@pProgmId+','+@pYearId+','+@PSemesterId+','+@pExamId+')
  
) as FailList pivot (max([UniversityRegNo]) for [Branch_Code] in ('+@cols1+')) as PVT'
--print @query1
execute(@query1)
end

--exec GetFailStudents 1,2,4,1



--SELECT ([StudentName]+' - '+convert(nvarchar(10),[FailCount])+' - '+Course+' - '+[StudentId_Rollno] +' - '+[YearOfStudy_Name] +' - '+[Section_Name]) As [UniversityRegNo]        
--,Branch_Code
--  FROM fGetFailStudents(1,2,3,1)



GO
/****** Object:  StoredProcedure [dbo].[GetMarkAvgDashBoard]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[GetMarkAvgDashBoard]
-- Add the parameters for the stored procedure here
	@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),
	@pSectionId NVARCHAR(50),
	@pExamId NVARCHAR(50),
	@pCourseid nvarchar(50) 	

AS BEGIN
--exec GetMarkAvgDashBoard 8,6,12,1,4,4
--set @pBranchId=8
--set @pYearOfStudyId=6
--set @pSemesterId=12
--set @pSectionId=3
--set @pExamId=1



DECLARE @cols1 AS NVARCHAR(MAX), @query1 AS NVARCHAR(MAX),@passmark nvarchar(5); 
set @passmark=(select SettingValue from T_CFG_CONFIGURATON_SETTINGS where SettingName='PassMark');
SET @cols1 = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_ShortCode)FROM  T_CFG_COURSE_MASTER CM
join T_MARK_ENTRY ME on CM.Course_Id=ME.Course_Id where ME.Branch_Id=@pBranchId and CM.Course_Id=@pCourseid and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId and ME.Section_Id=@pSectionId and ME.Exam_Id=@pExamId FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 


Select @cols1 SET @query1 = 'Select  [Desc],'+@cols1+' 
	from (Select ''No.of Students Appeared'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+'
	and SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0) as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


PRINT @query1

execute(@query1)

Select @cols1
 SET @query1 = 'Select  [Desc],'+@cols1+' 
	from (Select ''No.of Students Passed'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks>='+@passmark+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+'
	and SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0) as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)

Select @cols1 SET @query1 = 'Select   [Desc],'+@cols1+' 
	from (Select ''No.of Students Failed'' as [Desc], SP.StudentName +'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks<'+@passmark+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+'
	and SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0) as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)

Select @cols1 SET @query1 = 'Select  [Desc],'+@cols1+' 
	from (Select ''No.of Students Absent'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Is_Absent=1 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+'
	and SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0) as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)



end


--exec GetMarkAvgDashBoard  8,2,4,1,1,469

GO
/****** Object:  StoredProcedure [dbo].[GetMarkReport]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[GetMarkReport]
-- Add the parameters for the stored procedure here
	@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),
	@pSectionId NVARCHAR(50),
	@pExamId NVARCHAR(50),
	@pAcYrId NVARCHAR(50)

AS BEGIN

DECLARE @cols1 AS NVARCHAR(MAX), @query1 AS NVARCHAR(MAX); 
DECLARE @cols2 AS NVARCHAR(MAX), @query2 AS NVARCHAR(MAX); 
DECLARE @passMark NVARCHAR(5);

SET @passMark=(SELECT  SettingValue  FROM T_CFG_CONFIGURATON_SETTINGS WHERE SettingName='PassMark')

if(@pSectionId!=0)
begin
SET @cols1 = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_ShortCode)FROM  T_CFG_COURSE_MASTER CM
join T_MARK_ENTRY ME (NOLOCK) on CM.Course_Id=ME.Course_Id where ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId and ME.Section_Id=@pSectionId and ME.Exam_Id=@pExamId and ME.AcademicYear_Id=@pAcYrId FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 



Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Appeared'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where ME.Is_OD=0 and ME.Is_Absent=0 and SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 

	

execute(@query1)

Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Passed'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>='+@passMark+' and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Failed'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks<'+@passMark+' and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Absent'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Is_Absent=1 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)
 --No.of Students Secured ''S'' Grade(91-100)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured S Grade(91-100)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=91 and me.Marks<=100 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

 --No.of Students Secured ''A'' Grade(81-90)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured A Grade(81-90)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=81 and me.Marks<=90 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

 --No.of Students Secured ''B'' Grade(71-80)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured B Grade(71-80)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=71 and me.Marks<=80 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

--No.of Students Secured ''C'' Grade(61-70)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured C Grade(61-70)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=51 and me.Marks<=70 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

--No.of Students Secured ''D'' Grade(56-60)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured D Grade(56-60)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=56 and me.Marks<=60 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)


--No.of Students Secured ''E'' Grade(50-55)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured E Grade(50-55)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=50 and me.Marks<=55 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

--No.of Students Secured Grade(41-49)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured (41-49)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=41 and me.Marks<=49 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

--No.of Students Secured Grade(21-40)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured (21-40)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=21 and me.Marks<=40 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)


--No.of Students Secured <=20
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured <= 20'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks<=20 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

--Exam Date
Select @cols1 SET @query1 = 'Select  [Desc],  '+@cols1+' 
	from (select ''Date of Test'' as [Desc], CM.Course_ShortCode,convert(varchar(150),ed.Exam_Date,103) as Exam_Date  from T_EXAMDEFINITION_DETAIL ed 
	join T_EXAM_DEFINITION E on ed.Exam_Definition_Id = E.Exam_Definition_Id
join T_CFG_COURSE_MASTER CM on ed.Course_Id=CM.Course_Id 
	where  E.Academic_Year_Id='+@pAcYrId+' and E.Branch_Id='+@pBranchId+' and E.YearOfStudy_Id='+@pYearOfStudyId+' and E.Semester_Id='+@pSemesterId+' and E.Section_Id='+@pSectionId+' and E.Exam_Id='+@pExamId+')
	 as Source pivot(max(Exam_Date) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

--Exam Date of Submission

Select @cols1 SET @query1 = 'Select  [Desc],  '+@cols1+' 
	from (
	select distinct ''Date of Submission'' as [Desc], CM.Course_ShortCode,convert(varchar(150),ME.Created_Date,103) as Created_Date  from T_EXAMDEFINITION_DETAIL ed 
join T_EXAM_DEFINITION E on ed.Exam_Definition_Id = E.Exam_Definition_Id
join T_CFG_COURSE_MASTER CM on ed.Course_Id=CM.Course_Id 
join T_MARK_ENTRY ME (NOLOCK) on ed.Course_Id=ME.Course_Id 
where ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+'
	)
	 as Source pivot(max(Created_Date) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

--class avg

Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Absent'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Is_Absent=1 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

--pass percentage

Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Absent'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Is_Absent=1 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

SET @cols2 = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_Code),CONCAT('-', CM.Course_ShortCode) as CourseShortCode FROM  T_CFG_COURSE_MASTER CM
join T_MARK_ENTRY ME (NOLOCK) on CM.Course_Id=ME.Course_Id where ME.AcademicYear_Id=@pAcYrId and ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId and ME.Section_Id=@pSectionId and ME.Exam_Id=@pExamId
Order by CourseShortCode  FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 

select @cols2

end
else
begin

SET @cols1 = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_ShortCode)FROM  T_CFG_COURSE_MASTER CM
join T_MARK_ENTRY ME (NOLOCK) on CM.Course_Id=ME.Course_Id where  ME.AcademicYear_Id=@pAcYrId and ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId  and ME.Exam_Id=@pExamId FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 



Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Appeared'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 

	

execute(@query1)

Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Passed'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>='+@passMark+' and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Failed'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks<'+@passMark+' and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Absent'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Is_Absent=1 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)
 --No.of Students Secured ''S'' Grade(91-100)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured S Grade(91-100)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=91 and me.Marks<=100 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

 --No.of Students Secured ''A'' Grade(81-90)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured A Grade(81-90)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=81 and me.Marks<=90 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

 --No.of Students Secured ''B'' Grade(71-80)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured B Grade(71-80)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=71 and me.Marks<=80 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

--No.of Students Secured ''C'' Grade(61-70)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured C Grade(61-70)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=51 and me.Marks<=70 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

--No.of Students Secured ''D'' Grade(56-60)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured D Grade(56-60)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=56 and me.Marks<=60 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)


--No.of Students Secured ''E'' Grade(50-55)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured E Grade(50-55)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=50 and me.Marks<=55 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

--No.of Students Secured Grade(41-49)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured (41-49)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=41 and me.Marks<=49 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

--No.of Students Secured Grade(21-40)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured (21-40)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks>=21 and me.Marks<=40 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)


--No.of Students Secured <=20
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured <= 20'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Marks<=20 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

--Exam Date
Select @cols1 SET @query1 = 'Select  [Desc],  '+@cols1+' 
	from (select ''Date of Test'' as [Desc], CM.Course_ShortCode,convert(varchar(150),ed.Exam_Date,103) as Exam_Date  from T_EXAMDEFINITION_DETAIL ed 
	join T_EXAM_DEFINITION E on ed.Exam_Definition_Id = E.Exam_Definition_Id
join T_CFG_COURSE_MASTER CM on ed.Course_Id=CM.Course_Id 
	where E.Academic_Year_Id='+@pAcYrId+' and  E.Branch_Id='+@pBranchId+' and E.YearOfStudy_Id='+@pYearOfStudyId+' and E.Semester_Id='+@pSemesterId+'  and E.Exam_Id='+@pExamId+')
	 as Source pivot(max(Exam_Date) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

--Exam Date of Submission

Select @cols1 SET @query1 = 'Select  [Desc],  '+@cols1+' 
	from (
	select distinct ''Date of Submission'' as [Desc], CM.Course_ShortCode,convert(varchar(150),ME.Created_Date,103) as Created_Date  from T_EXAMDEFINITION_DETAIL ed 
join T_EXAM_DEFINITION E on ed.Exam_Definition_Id = E.Exam_Definition_Id
join T_CFG_COURSE_MASTER CM on ed.Course_Id=CM.Course_Id 
join T_MARK_ENTRY ME (NOLOCK) on ed.Course_Id=ME.Course_Id 
where ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+'
	)
	 as Source pivot(max(Created_Date) for Course_ShortCode in ('+@cols1+')) as PVT' 


execute(@query1)

--class avg

Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Absent'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Is_Absent=1 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)

--pass percentage

Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Absent'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_ShortCode  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where SP.Status=1 and SP.Delete_Flag=0 and SP.TC_ISSUED=0 and me.Is_Absent=1 and ME.AcademicYear_Id='+@pAcYrId+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+'  and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_ShortCode in ('+@cols1+')) as PVT' 



execute(@query1)


SET @cols2 = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_Code),CONCAT('-', CM.Course_ShortCode) as CourseShortCode FROM  T_CFG_COURSE_MASTER CM
join T_MARK_ENTRY ME (NOLOCK) on CM.Course_Id=ME.Course_Id where  ME.AcademicYear_Id=@pAcYrId and ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId  and ME.Exam_Id=@pExamId
Order by CourseShortCode FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 

select @cols2
end
end

--exec [GetMarkReport] 8,2,4,1,1,3

GO
/****** Object:  StoredProcedure [dbo].[GetMarksStatistics]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
--exec [GetMarksStatistics]'1','2','4','1','1'
CREATE Proc [dbo].[GetMarksStatistics]--'1','2','4','1','1'
@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),
	@pSectionId NVARCHAR(50),
	@pExamId NVARCHAR(50) 	
as begin



DECLARE @cols1 AS NVARCHAR(MAX), @query1 AS NVARCHAR(MAX); 
SET @cols1 = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_Code)FROM  T_CFG_COURSE_MASTER CM
join T_MARK_ENTRY ME on CM.Course_Id=ME.Course_Id where ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId and ME.Section_Id=@pSectionId and ME.Exam_Id=@pExamId FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 

 --No.of Students Secured ''S'' Grade(91-100)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured S Grade(91-100)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_Code  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks>=91 and me.Marks<=100 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_Code in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)

 --No.of Students Secured ''A'' Grade(81-90)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured A Grade(81-90)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_Code  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks>=81 and me.Marks<=90 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_Code in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)

 --No.of Students Secured ''B'' Grade(71-80)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured B Grade(71-80)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_Code  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks>=71 and me.Marks<=80 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_Code in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)

--No.of Students Secured ''C'' Grade(61-70)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured C Grade(61-70)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_Code  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks>=61 and me.Marks<=70 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_Code in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)

--No.of Students Secured ''D'' Grade(56-60)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured D Grade(56-60)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_Code  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks>=56 and me.Marks<=60 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_Code in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)


--No.of Students Secured ''E'' Grade(50-55)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured E Grade(50-55)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_Code  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks>=50 and me.Marks<=55 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_Code in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)

--No.of Students Secured Grade(41-49)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured (41-49)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_Code  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks>=41 and me.Marks<=49 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_Code in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)

--No.of Students Secured Grade(21-40)
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured (21-40)'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_Code  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks>=21 and me.Marks<=40 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_Code in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)


--No.of Students Secured <=20
Select @cols1 SET @query1 = 'Select  [Desc], '+@cols1+' 
	from (Select ''No.of Students Secured <= 20'' as [Desc], SP.StudentName+'' ''+SP.Initial as SNAME,CM.Course_Code  from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	where me.Marks<=20 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and ME.Exam_Id='+@pExamId+') as Source pivot(count(SNAME) for Course_Code in ('+@cols1+')) as PVT' 

PRINT @query1

execute(@query1)
end
GO
/****** Object:  StoredProcedure [dbo].[GetMorethan3daysConsecutiveLeaveReport]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


CREATE Procedure [dbo].[GetMorethan3daysConsecutiveLeaveReport]
@fromdate varchar(50),
@Todate varchar(50),
@branch varchar(50),
@year varchar(50)

As 
Begin

SELECT A.StudentTable_Id, A.StudentId_Rollno,A.StudentName,A.YearOfStudy_Id As Class,
RIGHT('0' + CAST(DATEPART(d, MIN(A.Attendance_Date)) AS NVARCHAR(50)), 2) + ' - ' + CONVERT(NVARCHAR(50), MAX(A.Attendance_Date), 103) AS Duration,
count(A.DiffCode) AS Count_Total
	from (select * from fncConsecutiveLeaveAttendanceReport(@fromdate,@Todate,@branch,@year)) AS A
	group by A.StudentTable_Id, A.StudentId_Rollno,A.StudentName,A.YearOfStudy_Id, A.DiffCode
	having count(A.DiffCode) >= 3
	order by A.YearOfStudy_Id
End


--exec [GetMorethan3daysConsecutiveLeaveReport] '2014-06-01','2014-08-24',8,3






GO
/****** Object:  StoredProcedure [dbo].[GetMorethan3daysConsecutiveLeaveReportNEW]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE Procedure [dbo].[GetMorethan3daysConsecutiveLeaveReportNEW]
@fromdate varchar(50),
@Todate varchar(50),
@branch varchar(50),
@year varchar(50),
@noofDays bigint

As 
Begin

SELECT A.StudentTable_Id, A.StudentId_Rollno,A.StudentName,A.YearOfStudy_Id As Class,
RIGHT('0' + CAST(DATEPART(d, MIN(A.Attendance_Date)) AS NVARCHAR(50)), 2) + ' - ' + CONVERT(NVARCHAR(50), MAX(A.Attendance_Date), 103) AS Duration,
count(A.DiffCode) AS Count_Total
	from (select * from fncConsecutiveLeaveAttendanceReport(@fromdate,@Todate,@branch,@year)) AS A
	group by A.StudentTable_Id, A.StudentId_Rollno,A.StudentName,A.YearOfStudy_Id, A.DiffCode
	having count(A.DiffCode) >= @noofDays
	order by A.YearOfStudy_Id
End


--exec [GetMorethan3daysConsecutiveLeaveReport] '2014-06-01','2014-08-24','8,1',3






GO
/****** Object:  StoredProcedure [dbo].[GetMorethan3daysLeaveReport]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

--exec [GetMorethan3daysLeaveReport] '2014-07-01','2014-07-31',8

Create Procedure [dbo].[GetMorethan3daysLeaveReport]
@fromdate varchar(50),
@Todate varchar(50),
@branch varchar(50),
@year varchar(50)

As 

Begin


 select A.StudentId_Rollno,A.StudentName,A.YearOfStudy_Id As TotLeave
,

--convert(nvarchar(max),(SELECT distinct ',' + QUOTENAME(CM.LeaveCount)FROM  fncAttendanceReport(@fromdate ,@Todate ,@branch,@year) CM
--	WHERE        A.StudentId_Rollno = CM.StudentId_Rollno
-- FOR XML PATH(''), TYPE) .value('.', 'NVARCHAR(MAX)') )as Course


Course= STUFF
                           ((SELECT        ',' + Convert(varchar(50),md.LeaveCount) As SumLeaveoddays
                              FROM           fncAttendanceReport(@fromdate ,@Todate ,@branch,@year)As  md
                               WHERE        A.StudentId_Rollno = md.StudentId_Rollno FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)'), 1, 1, '') 

from fncAttendanceReport(@fromdate ,@Todate ,@branch,@year) As A
group by A.StudentId_Rollno,A.StudentName,A.LeaveCount , A.YearOfStudy_Id,A.LeaveCount having  sum(A.lCount) >= 3
End


--exec [GetMorethan3daysLeaveReport] '2014-07-01','2014-07-31',8





GO
/****** Object:  StoredProcedure [dbo].[GetMorethan3daysLeaveReportNEW]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


--exec [GetMorethan3daysLeaveReport] '2014-07-01','2014-07-31',8

CREATE Procedure [dbo].[GetMorethan3daysLeaveReportNEW]
@fromdate varchar(50),
@Todate varchar(50),
@branch varchar(50),
@year varchar(50),
@noofDays bigint

As 

Begin


 select A.StudentId_Rollno,A.StudentName,A.YearOfStudy_Id As TotLeave
,

--convert(nvarchar(max),(SELECT distinct ',' + QUOTENAME(CM.LeaveCount)FROM  fncAttendanceReport(@fromdate ,@Todate ,@branch,@year) CM
--	WHERE        A.StudentId_Rollno = CM.StudentId_Rollno
-- FOR XML PATH(''), TYPE) .value('.', 'NVARCHAR(MAX)') )as Course


Course= STUFF
                           ((SELECT        ',' + Convert(varchar(50),md.LeaveCount) As SumLeaveoddays
                              FROM           fncAttendanceReportNEW(@fromdate ,@Todate ,@branch,@year)As  md
                               WHERE        A.StudentId_Rollno = md.StudentId_Rollno FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)'), 1, 1, '') 
							   

from fncAttendanceReportNEW(@fromdate ,@Todate ,@branch,@year) As A
group by A.StudentId_Rollno,A.StudentName,A.LeaveCount , A.YearOfStudy_Id,A.LeaveCount having  sum(A.lCount) >= @noofDays
End


--exec [GetMorethan3daysLeaveReportNEW] '2014-07-01','2014-07-31',8,2






GO
/****** Object:  StoredProcedure [dbo].[getStaffDetailbyCoursewise]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[getStaffDetailbyCoursewise]
@Branchid bigint,
@year bigint,
@section bigint




As
begin
SELECT        dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.AcademicYear_Id, dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.Branch_Id, 
                         dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.YearOfStudy_Id, dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.Semester_Id, 
                         dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.Staff_Id, dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.Id, dbo.T_STAFF_ASSIGINING_CLASS_COURSE_DTL.Course_Id, 
                         dbo.T_STAFF_ASSIGINING_CLASS_COURSE_DTL.Class_Section_Id, dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.Status, dbo.T_CFG_COURSE_MASTER.Course_Code, 
                         dbo.T_CFG_COURSE_MASTER.Course_Name, dbo.T_CFG_STAFF_MASTER.Staff_Code, dbo.T_CFG_STAFF_MASTER.Staff_Name
FROM            dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR INNER JOIN
                         dbo.T_STAFF_ASSIGINING_CLASS_COURSE_DTL ON dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.Id = dbo.T_STAFF_ASSIGINING_CLASS_COURSE_DTL.Hdr_Id INNER JOIN
                         dbo.T_CFG_COURSE_MASTER ON dbo.T_STAFF_ASSIGINING_CLASS_COURSE_DTL.Course_Id = dbo.T_CFG_COURSE_MASTER.Course_Id INNER JOIN
                         dbo.T_CFG_STAFF_MASTER ON dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.Staff_Id = dbo.T_CFG_STAFF_MASTER.Staff_Id
                      where  dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.Branch_Id=@Branchid and dbo.T_STAFF_ASSIGINING_CLASS_COURSE_HDR.YearOfStudy_Id=@year and dbo.T_STAFF_ASSIGINING_CLASS_COURSE_DTL.Class_Section_Id=@section


End
GO
/****** Object:  StoredProcedure [dbo].[GetStudentAUResult]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE proc [dbo].[GetStudentAUResult]
@Auresult varchar(50),
@publishyear varchar(50),
@resultType varchar(50),
@Branch varchar(50),
@yearofstudy varchar(50),
@semester varchar(50) 

as
begin
DECLARE @cols1 AS NVARCHAR(MAX), @query1 AS NVARCHAR(MAX); 
SET @cols1 = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_Code)FROM  T_CFG_COURSE_MASTER CM
join T_AUEXAM_RESULTS ME on CM.Course_Code=ME.Course_Code where  ME.Branch_Id=@Branch and ME.Semester_Id=@semester  FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 


Select @cols1 
SET @query1 = 'Select SNAME As [Student Name], '+@cols1+' 
	from (Select  SP.StudentName+'' ''+SP.Initial as SNAME,ME.Grade,CM.Course_Code as Course_Code from  T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_AUEXAM_RESULTS ME on SP.UniversityRegNo=ME.Student_Registration_No
	join T_CFG_COURSE_MASTER CM on ME.Course_Code=CM.Course_Code where ME.Branch_Id='''+@Branch+'''  and ME.Semester_Id='''+@semester+''' and ME.AUResult='''+@Auresult+'''	and ME.[Result_Published_Year_Id]='''+@publishyear+''' and ME.[Result_Type_Id]='''+@resultType+'''
	) as Source pivot(max(Grade) for Course_Code in ('+@cols1+')) as PVT' 

	--print @query1;
execute(@query1)

end

--exec GetStudentAUResult  'Regular',1,5,8,2,4

--select * from T_AUEXAM_RESULTS where branch_id=8 and semester_id=4
GO
/****** Object:  StoredProcedure [dbo].[GetStudentExamPerformance]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[GetStudentExamPerformance]
@StudentTableId NVARCHAR(50),
@pSemId nvarchar(5)
AS BEGIN

DECLARE 
@cols1 NVARCHAR(MAX),
@query1 NVARCHAR(MAX),
@query2 NVARCHAR(MAX),
@examid NVARCHAR(MAX),
@vYearId nvarchar(5),
@vSecId nvarchar(5),
@pAcYrId nvarchar(5)

set @vYearId=(select YearOfStudy_Id from T_CFG_SEMESTER where Semester_Id=@pSemId);

set @vSecId=(
select top 1 SECTION_ID  from T_MARK_ENTRY (NOLOCK)
where StudentTable_Id=@StudentTableId and SEMESTER_ID=@pSemId order by Mark_Entry_Id desc
);

set @pAcYrId=(
select top 1 AcademicYear_Id  from T_MARK_ENTRY (NOLOCK)
where StudentTable_Id=@StudentTableId and SEMESTER_ID=@pSemId order by Mark_Entry_Id desc
);

--SET @cols1 = STUFF((SELECT ',' + QUOTENAME(Exam_Name) FROM T_CFG_EXAM_MASTER EM WHERE Delete_Flag = 0 ORDER BY Order_By 
--					FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '')
SET @examid = (SELECT YE.AvailableExam_Ids from T_CFG_YEAROFSTUDY_MASTER YE								
				where YE.YearOfStudy_Id=@vYearId )

SET @cols1 =STUFF((select   ',' + QUOTENAME( E.Exam_Name)  from T_MARK_ENTRY M  (NOLOCK)
				   join T_CFG_EXAM_MASTER  E on M.Exam_Id=E.Exam_Id
				   --join T_CFG_SEMESTER SEM on M.Semester_Id = SEM.Semester_Id
					WHERE M.Semester_Id=@pSemId and M.AcademicYear_Id=@pAcYrId and E.Exam_Id in (select s from SplitString(@examid,','))  group by E.Exam_Name,M.Exam_Id order by M.Exam_Id
					 FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '')

SELECT @query1 = 'SELECT Course_Name AS [NAME OF THE SUBJECT], Course_Code AS [SUBJECT CODE], '+@cols1+' FROM
(SELECT ME.StudentTable_Id, CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name,
 --CAST(ME.Marks AS BIGINT) AS Marks,
 case when (me.Is_Absent=1 and (me.marks=''0'' or me.Marks=''00'' or me.Marks=''000'')) then ''AB''  when (me.Is_OD=1 and (me.marks=''0'' or me.Marks=''00'' or me.Marks=''000'')) then ''OD'' else ME.Marks end as [marksPivot]
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = '+ @StudentTableId +'
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = '+@vYearId+' and ME.Semester_Id = '+@pSemId+') AS SRC
		 PIVOT (MAX([marksPivot]) FOR Exam_Name IN (' + @cols1 + ' )) AS PVT 
		
SELECT Course_Name AS [NAME OF THE SUBJECT], Course_Code AS [SUBJECT CODE], '+@cols1+' FROM
(SELECT ME.StudentTable_Id, CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name,
 --CAST(ME.Marks AS BIGINT) AS Marks,
 case when (me.Is_Absent_Retest=1 and (me.RetestMarks=''0'' or me.RetestMarks=''00'' or me.RetestMarks=''000'')) then ''AB''  
 when (me.Is_OD=1 and (me.RetestMarks=''0'' or me.RetestMarks=''00'' or me.RetestMarks=''000'')) then ''OD'' else ME.RetestMarks end as [retestMarksPivot]
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = '+ @StudentTableId +'
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = '+@vYearId+' and ME.Semester_Id = '+@pSemId+') AS SRC
		 PIVOT (MAX([retestMarksPivot]) FOR Exam_Name IN (' + @cols1 + ' )) AS PVT

SELECT  ''Total'','''', ' + @cols1 + ' FROM
(SELECT  EM.Exam_Name, 
 CAST(ME.Marks AS BIGINT) AS Marks
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = '+ @StudentTableId +'
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = '+@vYearId+' and ME.Semester_Id = '+@pSemId+') AS SRC
		 PIVOT (SUM(Marks) FOR Exam_Name IN (' + @cols1 + ' )) AS PVT 
		 
		

SELECT ''Attendance Percentage'','''',' + @cols1 + ' FROM
(SELECT (SUM(CASE AE.Attendance_Status WHEN ''P'' THEN 1 ELSE 0 END) * 100) / (SUM(CASE AE.Attendance_Status WHEN ''P'' THEN 1 ELSE 0 END) + SUM(CASE AE.Attendance_Status WHEN ''AB'' THEN 1 ELSE 0 END)) AS [Percentage],
		 EM.Exam_Name
		 FROM T_ATTENDANCEENTRY AE (NOLOCK)
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S on AE.StudentTable_Id=S.Id
		 JOIN T_EXAM_DEFINITION ED on ED.Exam_Id in (' + @examid + ')
		 JOIN T_CFG_EXAM_MASTER EM on ED.Exam_Id = EM.Exam_Id
		 WHERE  ED.Academic_Year_Id='+@pAcYrId+' and S.Branch_Id = ED.Branch_Id and  ED.YearOfStudy_Id='+@vYearId+' and
		 S.Semester_Id = '+@pSemId+' and ED.Section_Id='+@vSecId+' and
		 AE.Attendance_Date between ED.Attendance_FromDate and ED.Attendance_ToDate and AE.StudentTable_Id='+ @StudentTableId +'
		 group by ED.Exam_Id, EM.Exam_Name) AS SRC
		 PIVOT (MAX([Percentage]) FOR Exam_Name IN (' + @cols1 + ')) AS PVT'

		 --select @query1
		-- select @cols1
EXEC(@query1) 


SELECT S.StudentName+' '+S.Initial as StudentName, S.StudentId_Rollno,S.UniversityRegNo,S.SNo, B.Branch_Code, SEC.Section_Name, Y.YearOfStudy_Name, P.Programme_Name, SEM.Semester_Name, S.FatherName,
	   S.PermenentAddress, S.City, S.Pincode	 FROM T_CFG_STUDENT_PERSONAL_DETAIL S
		 JOIN T_CFG_BRANCH B ON S.Branch_Id = B.Branch_Id
		 JOIN T_CFG_YEAROFSTUDY_MASTER Y ON S.YearOfStudy_Id = Y.YearOfStudy_Id
		 JOIN T_CFG_SEMESTER SEM ON S.Semester_Id = SEM.Semester_Id
		 JOIN T_CFG_SECTION_MASTER SEC ON S.Section_Id = SEC.ID
		 JOIN T_CFG_PROGRAMME P ON B.Programme_Id = P.Programme_Id
		 WHERE S.Id = @StudentTableId	
END


--exec GetStudentExamPerformance 1135,5
GO
/****** Object:  StoredProcedure [dbo].[GetStudentMarkForAllExam]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[GetStudentMarkForAllExam]
@StudentTableId NVARCHAR(50),
@pSemId nvarchar(5)
AS BEGIN

DECLARE 
@cols1 NVARCHAR(MAX),
@query1 NVARCHAR(MAX),
@query2 NVARCHAR(MAX),
@examid NVARCHAR(MAX),
@vYearId nvarchar(5),
@vSecId nvarchar(5),
@pAcYrId nvarchar(5)

set @vYearId=(select YearOfStudy_Id from T_CFG_SEMESTER where Semester_Id=@pSemId);

set @vSecId=(
select top 1 SECTION_ID  from T_MARK_ENTRY (NOLOCK)
where StudentTable_Id=@StudentTableId and SEMESTER_ID=@pSemId order by Mark_Entry_Id desc
);

set @pAcYrId=(
select top 1 AcademicYear_Id  from T_MARK_ENTRY (NOLOCK)
where StudentTable_Id=@StudentTableId and SEMESTER_ID=@pSemId order by Mark_Entry_Id desc
);

--SET @cols1 = STUFF((SELECT ',' + QUOTENAME(Exam_Name) FROM T_CFG_EXAM_MASTER EM WHERE Delete_Flag = 0 ORDER BY Order_By 
--					FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '')
SET @examid = (SELECT YE.AvailableExam_Ids from T_CFG_YEAROFSTUDY_MASTER YE								
				where YE.YearOfStudy_Id=@vYearId )

SET @cols1 =STUFF((select   ',' + QUOTENAME( E.Exam_Name)  from T_MARK_ENTRY M  (NOLOCK)
				   join T_CFG_EXAM_MASTER  E on M.Exam_Id=E.Exam_Id
				   --join T_CFG_SEMESTER SEM on M.Semester_Id = SEM.Semester_Id
					WHERE M.Semester_Id=@pSemId and M.AcademicYear_Id=@pAcYrId and E.Exam_Id in (select s from SplitString(@examid,','))  group by E.Exam_Name,M.Exam_Id order by M.Exam_Id
					 FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '')

SELECT @query1 = 'SELECT Course_Name AS [NAME OF THE SUBJECT], Course_Code AS [SUBJECT CODE], '+@cols1+' FROM
(SELECT ME.StudentTable_Id, CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name,
 --CAST(ME.Marks AS BIGINT) AS Marks,
 case when (me.Is_Absent=1 and (me.marks=''0'' or me.Marks=''00'' or me.Marks=''000'')) then ''AB''  when (me.Is_OD=1 and (me.marks=''0'' or me.Marks=''00'' or me.Marks=''000'')) then ''OD'' else ME.Marks end as [marksPivot]
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = '+ @StudentTableId +'
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = '+@vYearId+' and ME.Semester_Id = '+@pSemId+') AS SRC
		 PIVOT (MAX([marksPivot]) FOR Exam_Name IN (' + @cols1 + ' )) AS PVT 
		
		

SELECT  ''Total'','''', ' + @cols1 + ' FROM
(SELECT  EM.Exam_Name, 
 CAST(ME.Marks AS BIGINT) AS Marks
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = '+ @StudentTableId +'
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = '+@vYearId+' and ME.Semester_Id = '+@pSemId+') AS SRC
		 PIVOT (SUM(Marks) FOR Exam_Name IN (' + @cols1 + ' )) AS PVT 
		 
		

SELECT ''Attendance Percentage'','''',' + @cols1 + ' FROM
(SELECT (SUM(CASE AE.Attendance_Status WHEN ''P'' THEN 1 ELSE 0 END) * 100) / (SUM(CASE AE.Attendance_Status WHEN ''P'' THEN 1 ELSE 0 END) + SUM(CASE AE.Attendance_Status WHEN ''AB'' THEN 1 ELSE 0 END)) AS [Percentage],
		 EM.Exam_Name
		 FROM T_ATTENDANCEENTRY AE (NOLOCK)
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S on AE.StudentTable_Id=S.Id
		 JOIN T_EXAM_DEFINITION ED on ED.Exam_Id in (' + @examid + ')
		 JOIN T_CFG_EXAM_MASTER EM on ED.Exam_Id = EM.Exam_Id
		 WHERE  ED.Academic_Year_Id='+@pAcYrId+' and S.Branch_Id = ED.Branch_Id and  ED.YearOfStudy_Id='+@vYearId+' and
		 S.Semester_Id = '+@pSemId+' and ED.Section_Id='+@vSecId+' and
		 AE.Attendance_Date between ED.Attendance_FromDate and ED.Attendance_ToDate and AE.StudentTable_Id='+ @StudentTableId +'
		 group by ED.Exam_Id, EM.Exam_Name) AS SRC
		 PIVOT (MAX([Percentage]) FOR Exam_Name IN (' + @cols1 + ')) AS PVT'

		 --select @query1
		-- select @cols1
EXEC(@query1) 


SELECT S.StudentName+' '+S.Initial as StudentName, S.StudentId_Rollno,S.UniversityRegNo,S.SNo, B.Branch_Code, SEC.Section_Name, Y.YearOfStudy_Name, P.Programme_Name, SEM.Semester_Name, S.FatherName,
	   S.PermenentAddress, S.City, S.Pincode	 FROM T_CFG_STUDENT_PERSONAL_DETAIL S
		 JOIN T_CFG_BRANCH B ON S.Branch_Id = B.Branch_Id
		 JOIN T_CFG_YEAROFSTUDY_MASTER Y ON S.YearOfStudy_Id = Y.YearOfStudy_Id
		 JOIN T_CFG_SEMESTER SEM ON S.Semester_Id = SEM.Semester_Id
		 JOIN T_CFG_SECTION_MASTER SEC ON S.Section_Id = SEC.ID
		 JOIN T_CFG_PROGRAMME P ON B.Programme_Id = P.Programme_Id
		 WHERE S.Id = @StudentTableId	
END


--exec GetStudentMarkForAllExam 1135,5
GO
/****** Object:  StoredProcedure [dbo].[GetStudentMarkForExam]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[GetStudentMarkForExam]
@pStudentTableId NVARCHAR(50),
@pSemId NVARCHAR(5),
@pExamId NVARCHAR(5)
as
begin

DECLARE 
@cols1 NVARCHAR(MAX),
@query1 NVARCHAR(MAX),
@query2 NVARCHAR(MAX),
@yearId NVARCHAR(5),
@sectionId NVARCHAR(5),
@academicYearId NVARCHAR(5),
@passMark NVARCHAR(5)

--set @pStudentTableId = 1573;
--set @pSemId = 4;
--set @pExamId = 1;
set @passMark=(select SettingValue from T_CFG_CONFIGURATON_SETTINGS where SettingName='PassMark');

set @yearId = (select YearOfStudy_Id from T_CFG_SEMESTER where Semester_Id=@pSemId);

set @sectionId = (select top 1 SECTION_ID  from T_MARK_ENTRY  (NOLOCK)
					where StudentTable_Id=@pStudentTableId and SEMESTER_ID=@pSemId order by Mark_Entry_Id desc);

set @academicYearId=(select top 1 AcademicYear_Id  from T_MARK_ENTRY  (NOLOCK)
						where StudentTable_Id=@pStudentTableId and SEMESTER_ID=@pSemId order by Mark_Entry_Id desc);

SET @cols1 =STUFF((select   ',' + QUOTENAME( E.Exam_Name)  from T_MARK_ENTRY M  (NOLOCK)
					join T_CFG_EXAM_MASTER  E on M.Exam_Id=E.Exam_Id
					WHERE M.Semester_Id=@pSemId and M.AcademicYear_Id=@academicYearId and E.Exam_Id = @pExamId  
					group by E.Exam_Name,M.Exam_Id order by M.Exam_Id
					FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '')

SELECT @query1 = 'SELECT Course_Name AS [NAME OF THE SUBJECT], Course_Code AS [SUBJECT CODE], '+@cols1+', RESULT, [RETEST MARKS] FROM
(SELECT ME.StudentTable_Id, CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name,
 --CAST(ME.Marks AS BIGINT) AS Marks,
 case when (me.Is_Absent=1 and (me.marks=''0'' or me.Marks=''00'' or me.Marks=''000'')) then ''AB''  when (me.Is_OD=1 and (me.marks=''0'' or me.Marks=''00'' or me.Marks=''000'')) then ''OD'' else ME.Marks end as [marksPivot],
 case when me.Marks < ' + @passMark + ' then ''FAIL'' else ''PASS'' end as RESULT, 
 case when (me.Is_Absent_Retest=1 and (me.marks=''0'' or me.Marks=''00'' or me.Marks=''000'')) then ''AB'' else ISNULL(ME.RetestMarks,'''') end as [RETEST MARKS]		
		 FROM T_MARK_ENTRY ME  (NOLOCK)
		 JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = '+ @pStudentTableId +'
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = '+@yearId+' and ME.Semester_Id = '+@pSemId+' and ME.Exam_Id = '+@pExamId+') AS SRC
		 PIVOT (MAX([marksPivot]) FOR Exam_Name IN (' + @cols1 + ' )) AS PVT	

SELECT  ''Total'','''', ' + @cols1 + ','''','''' FROM
(SELECT  EM.Exam_Name, 
 CAST(ME.Marks AS BIGINT) AS Marks
		 FROM T_MARK_ENTRY ME  (NOLOCK)	
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = '+ @pStudentTableId +'
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = '+@yearId+' and ME.Semester_Id = '+@pSemId+') AS SRC
		 PIVOT (SUM(Marks) FOR Exam_Name IN (' + @cols1 + ' )) AS PVT

SELECT ''Attendance Percentage'','''',' + @cols1 + ','''','''' FROM
(SELECT (SUM(CASE AE.Attendance_Status WHEN ''P'' THEN 1 ELSE 0 END) * 100) / (SUM(CASE AE.Attendance_Status WHEN ''P'' THEN 1 ELSE 0 END) + SUM(CASE AE.Attendance_Status WHEN ''AB'' THEN 1 ELSE 0 END)) AS [Percentage],
		 EM.Exam_Name
		 FROM T_ATTENDANCEENTRY AE (NOLOCK)
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S on AE.StudentTable_Id=S.Id
		 JOIN T_EXAM_DEFINITION ED on ED.Exam_Id in (' + @pExamId + ')
		 JOIN T_CFG_EXAM_MASTER EM on ED.Exam_Id = EM.Exam_Id
		 WHERE  ED.Academic_Year_Id='+@academicYearId+' and S.Branch_Id = ED.Branch_Id and  ED.YearOfStudy_Id='+@yearId+' and
		 S.Semester_Id = '+@pSemId+' and ED.Section_Id='+@sectionId+' and
		 AE.Attendance_Date between ED.Attendance_FromDate and ED.Attendance_ToDate and AE.StudentTable_Id='+ @pStudentTableId +'
		 group by ED.Exam_Id, EM.Exam_Name) AS SRC
		 PIVOT (MAX([Percentage]) FOR Exam_Name IN (' + @cols1 + ')) AS PVT'
EXEC(@query1) 

SELECT S.StudentName+' '+S.Initial as StudentName, S.StudentId_Rollno,S.UniversityRegNo,S.SNo, B.Branch_Code, SEC.Section_Name, Y.YearOfStudy_Name, P.Programme_Name, SEM.Semester_Name, S.FatherName,
	   S.PermenentAddress, S.City, S.Pincode	 FROM T_CFG_STUDENT_PERSONAL_DETAIL S
		 JOIN T_CFG_BRANCH B ON S.Branch_Id = B.Branch_Id
		 JOIN T_CFG_YEAROFSTUDY_MASTER Y ON S.YearOfStudy_Id = Y.YearOfStudy_Id
		 JOIN T_CFG_SEMESTER SEM ON S.Semester_Id = SEM.Semester_Id
		 JOIN T_CFG_SECTION_MASTER SEC ON S.Section_Id = SEC.ID
		 JOIN T_CFG_PROGRAMME P ON B.Programme_Id = P.Programme_Id
		 WHERE S.Id = @pStudentTableId	
end

--exec GetStudentMarkForExam 1573,4,1
GO
/****** Object:  StoredProcedure [dbo].[GetStudentMarkForExamReport]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[GetStudentMarkForExamReport]-- '1135','6'
@StudentTableId NVARCHAR(50),
@pSemId nvarchar(5)
AS BEGIN

DECLARE 
@cols1 NVARCHAR(MAX),
@cols2 NVARCHAR(MAX),
@query1 NVARCHAR(MAX),
@query2 NVARCHAR(MAX),
@examid NVARCHAR(MAX),
@vYearId nvarchar(5),
@vSecId nvarchar(5),
@pAcYrId nvarchar(5),
@pBranchId nvarchar(5)

set @pBranchId=(
select top 1 Branch_ID  from T_MARK_ENTRY (NOLOCK)
where StudentTable_Id=@StudentTableId and SEMESTER_ID=@pSemId order by Mark_Entry_Id desc
);

set @vYearId=(select YearOfStudy_Id from T_CFG_SEMESTER where Semester_Id=@pSemId);

set @vSecId=(
select top 1 SECTION_ID  from T_MARK_ENTRY (NOLOCK)
where StudentTable_Id=@StudentTableId and SEMESTER_ID=@pSemId order by Mark_Entry_Id desc
);

set @pAcYrId=(
select top 1 AcademicYear_Id  from T_MARK_ENTRY (NOLOCK)
where StudentTable_Id=@StudentTableId and SEMESTER_ID=@pSemId order by Mark_Entry_Id desc
);

--SET @cols1 = STUFF((SELECT ',' + QUOTENAME(Exam_Name) FROM T_CFG_EXAM_MASTER EM WHERE Delete_Flag = 0 ORDER BY Order_By 
--					FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '')
SET @examid = (SELECT YE.AvailableExam_Ids from T_CFG_YEAROFSTUDY_MASTER YE								
				where YE.YearOfStudy_Id=@vYearId )

SET @cols1 =STUFF((select   ',' + QUOTENAME( E.Exam_Name)  from T_MARK_ENTRY M  (NOLOCK)
				   join T_CFG_EXAM_MASTER  E on M.Exam_Id=E.Exam_Id
				   --join T_CFG_SEMESTER SEM on M.Semester_Id = SEM.Semester_Id
					WHERE M.Semester_Id=@pSemId and M.AcademicYear_Id=@pAcYrId and E.Exam_Id in (select s from SplitString(@examid,','))  group by E.Exam_Name,M.Exam_Id order by M.Exam_Id
					 FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '')

select @cols2=coalesce(@cols2 + ', ', '')+(B.s+'.'+A.s + case B.s when 'R' then (' as [RT ' + cast((A.zeroBasedOccurance+1)as varchar) +']')
when 'A' then (' as [CLASS AVG ' + cast((A.zeroBasedOccurance+1)as varchar) +']') else ' as '+
(case A.s when '[UNIT TEST I]' then '[UT 1]' when '[UNIT TEST II]' then '[UT 2]' else A.s end) end) from SplitString((select @cols1),',') A, 
SplitString('M,R,A',',') B

SELECT @query1 = 'with tempMark([NAME OF THE SUBJECT], [SUBJECT CODE], '+@cols1+') as
(SELECT Course_Name AS [NAME OF THE SUBJECT], Course_Code AS [SUBJECT CODE], '+@cols1+' FROM
(SELECT ME.StudentTable_Id, CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name,
 --CAST(ME.Marks AS BIGINT) AS Marks,
 case when (me.Is_Absent=1 and (me.marks=''0'' or me.Marks=''00'' or me.Marks=''000'')) then ''AB''  when (me.Is_OD=1 and (me.marks=''0'' or me.Marks=''00'' or me.Marks=''000'')) then ''OD'' else ME.Marks end as [marksPivot]
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = '+ @StudentTableId +'
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = '+@vYearId+' and ME.Semester_Id = '+@pSemId+') AS SRC
		 PIVOT (MAX([marksPivot]) FOR Exam_Name IN (' + @cols1 + ' )) AS PVT), 

tempRetestMark([NAME OF THE SUBJECT], [SUBJECT CODE], '+@cols1+') as
(SELECT Course_Name AS [NAME OF THE SUBJECT], Course_Code AS [SUBJECT CODE], '+@cols1+' FROM
(SELECT ME.StudentTable_Id, CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name,
 --CAST(ME.Marks AS BIGINT) AS Marks,
 case when (me.Is_Absent_Retest=1 and (me.RetestMarks=''0'' or me.RetestMarks=''00'' or me.RetestMarks=''000'')) then ''AB'' else ISNULL(ME.RetestMarks,'''') end as [retestMarksPivot]
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = '+ @StudentTableId +'
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = '+@vYearId+' and ME.Semester_Id = '+@pSemId+') AS SRC
		 PIVOT (MAX([retestMarksPivot]) FOR Exam_Name IN (' + @cols1 + ' )) AS PVT),
	
tempAverage([NAME OF THE SUBJECT], [SUBJECT CODE], '+@cols1+') as		
(SELECT Course_Name AS [NAME OF THE SUBJECT], Course_Code AS [SUBJECT CODE], '+@cols1+' FROM
(SELECT CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name,
round(SUM(CAST(ME.Marks AS float))/SUM(case Is_absent when 1 then 0 else 1 end),2) as [avg]
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 --WHERE ME.StudentTable_Id = '+ @StudentTableId +'
		 and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id = '+@vYearId+' and ME.Semester_Id = '+@pSemId+' and me.Section_Id='+@vSecId+'
		 and ME.academicyear_id='+@pAcYrId+'
		 group by CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name
		 ) AS SRC
		 PIVOT (MAX([avg]) FOR Exam_Name IN ('+@cols1+')) AS PVT)
		 
Select M.[NAME OF THE SUBJECT], M.[SUBJECT CODE], '+@cols2+' from tempMark M join
tempRetestMark R on M.[SUBJECT CODE] = R.[SUBJECT CODE] join
tempAverage A on M.[SUBJECT CODE] = A.[SUBJECT CODE]'	

SELECT @query2 = 'SELECT  ''Total'', '''', ' + @cols1 + ' FROM
(SELECT  EM.Exam_Name, 
 CAST(ME.Marks AS BIGINT) AS Marks
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = '+ @StudentTableId +'
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = '+@vYearId+' and ME.Semester_Id = '+@pSemId+') AS SRC
		 PIVOT (SUM(Marks) FOR Exam_Name IN (' + @cols1 + ' )) AS PVT 
		 
		

SELECT ''Attendance Percentage'','''',' + @cols1 + ' FROM
(SELECT (SUM(CASE AE.Attendance_Status WHEN ''P'' THEN 1 ELSE 0 END) * 100) / (SUM(CASE AE.Attendance_Status WHEN ''P'' THEN 1 ELSE 0 END) + SUM(CASE AE.Attendance_Status WHEN ''AB'' THEN 1 ELSE 0 END)) AS [Percentage],
		 EM.Exam_Name
		 FROM T_ATTENDANCEENTRY AE (NOLOCK)
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S on AE.StudentTable_Id=S.Id
		 JOIN T_EXAM_DEFINITION ED on ED.Exam_Id in (' + @examid + ')
		 JOIN T_CFG_EXAM_MASTER EM on ED.Exam_Id = EM.Exam_Id
		 WHERE  ED.Academic_Year_Id='+@pAcYrId+' and ED.Branch_Id = S.Branch_Id and  ED.YearOfStudy_Id='+@vYearId+' and
		 ED.Semester_Id = '+@pSemId+' and ED.Section_Id='+@vSecId+' and
		 AE.Attendance_Date between ED.Attendance_FromDate and ED.Attendance_ToDate and AE.StudentTable_Id='+ @StudentTableId +'
		 group by ED.Exam_Id, EM.Exam_Name) AS SRC
		 PIVOT (MAX([Percentage]) FOR Exam_Name IN (' + @cols1 + ')) AS PVT
		 
SELECT ''Attendance Date'','''',' + @cols1 + ' FROM
(SELECT convert(nvarchar(50),ED.Attendance_FromDate,103) + '' - '' + convert(nvarchar(50),ED.Attendance_ToDate,103) AS [aDate],
		 EM.Exam_Name
		 FROM T_ATTENDANCEENTRY AE (NOLOCK)
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S on AE.StudentTable_Id=S.Id
		 JOIN T_EXAM_DEFINITION ED on ED.Exam_Id in (' + @examid + ')
		 JOIN T_CFG_EXAM_MASTER EM on ED.Exam_Id = EM.Exam_Id
		 WHERE  ED.Academic_Year_Id='+@pAcYrId+' and ED.Branch_Id = S.Branch_Id and  ED.YearOfStudy_Id='+@vYearId+' and
		 ED.Semester_Id = '+@pSemId+' and ED.Section_Id='+@vSecId+' and
		 AE.Attendance_Date between ED.Attendance_FromDate and ED.Attendance_ToDate and AE.StudentTable_Id='+ @StudentTableId +'
		 group by ED.Exam_Id, EM.Exam_Name,ED.Attendance_FromDate, ED.Attendance_ToDate) AS SRC
		 PIVOT (MAX([aDate]) FOR Exam_Name IN (' + @cols1 + ')) AS PVT'

EXEC(@query1) 
EXEC(@query2) 

SELECT S.StudentName+' '+S.Initial as StudentName, S.StudentId_Rollno,S.UniversityRegNo,S.SNo, B.Branch_Code, B.Branch_Name, SEC.Section_Name, Y.YearOfStudy_Name, P.Programme_Name, SEM.Semester_Name, S.FatherName,
	   S.PermenentAddress, S.City, S.Pincode	 FROM T_CFG_STUDENT_PERSONAL_DETAIL S
		 JOIN T_CFG_BRANCH B ON S.Branch_Id = B.Branch_Id
		 JOIN T_CFG_YEAROFSTUDY_MASTER Y ON S.YearOfStudy_Id = Y.YearOfStudy_Id
		 JOIN T_CFG_SEMESTER SEM ON S.Semester_Id = SEM.Semester_Id
		 JOIN T_CFG_SECTION_MASTER SEC ON S.Section_Id = SEC.ID
		 JOIN T_CFG_PROGRAMME P ON B.Programme_Id = P.Programme_Id
		 WHERE S.Id = @StudentTableId	
END


--exec GetStudentMarkForExamReport 1135,7

GO
/****** Object:  StoredProcedure [dbo].[GetStudentMarkList]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[GetStudentMarkList]
-- Add the parameters for the stored procedure here
	@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),
	@pSectionId NVARCHAR(50),
	@pExamId NVARCHAR(50),
	@pfromDt NVARCHAR(50),
	@pToDt nvarchar(50),
	@pOrderById bigint,
	@pAcYr NVARCHAR(50)

AS BEGIN
DECLARE @cols AS NVARCHAR(MAX), @query AS NVARCHAR(MAX), @pOrderByName AS NVARCHAR(50),@pAppText NVARCHAR(MAX); --,@pAcYr NVARCHAR(50); 

--SET @pAcYr=(select ID from T_CFG_ACADEMIC_YEAR where Status=1 and flag=0 and Academic_year_Status=1)

SET @pOrderByName=(select ORDER_BY_VALUE from T_CFG_ORDER_BY_DETAILS where ID=@pOrderById)

if(@pOrderByName='UniversityRegNo')
begin

SET @pAppText=' ORDER BY Section,[Register No]'

end
else if(@pOrderByName='StudentId_Rollno')
begin
SET @pAppText=' ORDER BY Section,[Student Id]';
end
else if(@pOrderByName='StudentName')
begin
SET @pAppText=' ORDER BY Section,[Student Name]';
end
else if(@pOrderByName='SNo')
begin
SET @pAppText=' ORDER BY Section,[SNO]';
end
else if(@pOrderByName='Male')
begin
SET @pAppText=' ORDER BY Section,[Gender] DESC';
end
else if(@pOrderByName='Female')
begin
SET @pAppText=' ORDER BY Section,[Gender]';
end

if(@pOrderById=75)
begin
SET @pAppText=' ORDER BY Section,[Rank]';
end




if(@pSectionId!=0)

begin
SET @cols = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_ShortCode) FROM T_CFG_COURSE_MASTER CM
join T_MARK_ENTRY ME (NOLOCK) on CM.Course_Id=ME.Course_Id where ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId and ME.Section_Id=@pSectionId and ME.Exam_Id=@pExamId and ME.AcademicYear_Id=@pAcYr FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 


select @cols 
SET @query = 'Select StuTabId,Gender,SNo as [SNO],StudentId_Rollno  as [Student Id], UniversityRegNo as [Register No], 
 StudentName as [Student Name],Section, '+@cols+' ,Total,Rank,[Attendance Percentage]
	from (
	Select SP.Id as StuTabId,SP.Gender as [Gender], SP.SNo, SP.StudentId_Rollno,SP.UniversityRegNo,SP.StudentName+'' ''+SP.Initial as StudentName,CM.Course_ShortCode,
	(case when (ME.Is_Absent=1 and ME.Marks=0) then ''AB'' when (ME.Is_OD=1 and ME.Marks=0) then ''OD'' else ME.Marks  end) as MARKS ,
	 SC.Section_Name as Section,f.Total,f.rank_No as Rank,f.[Attendance Percentage] 
	 from T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	join T_CFG_SECTION_MASTER SC on SP.Section_Id=SC.id
	join fnTotalAndAttendanceWithSection('+@pBranchId+','+@pYearOfStudyId+','+@pSemesterId+','+@pSectionId+','+@pExamId+',
										 '''+@pfromDt+''','''+@pToDt+''','+@pAcYr+') as f on sp.Id=f.StuTabId
	where ME.AcademicYear_Id='+@pAcYr+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and 
		  ME.Semester_Id='+@pSemesterId+' and ME.Section_Id='+@pSectionId+' and
		  ME.Exam_Id='+@pExamId+' and SP.Delete_Flag=0 and SP.Status=1 and SP.TC_ISSUED=0

	
	) as Source pivot(MAX(MARKS) for Course_ShortCode in ('+@cols+')) as PVT
	'+@pAppText+' '

	--select * from T_CFG_STUDENT_PERSONAL_DETAIL SP where SP.Delete_Flag=0 and SP.Status=1 and SP.TC_ISSUED=0
 
PRINT @query

execute(@query)

select distinct cm.Course_Code as [Sub Code],cm.Course_Name as [Subjet Name],sm.Staff_Name as [Staff Name] from T_COURSE_REGULATION_SYALLBUS_DTL rd 
join T_COURSE_REGULATION_SYALLBUS_HDR rh on rd.ID = rh.ID
join T_CFG_COURSE_MASTER cm on rd.Course_Code=cm.Course_Id
join T_STAFF_ASSIGINING_CLASS_COURSE_DTL sd on cm.Course_Id=sd.Course_Id
join T_STAFF_ASSIGINING_CLASS_COURSE_HDR sh on sd.Hdr_Id=sh.Id
left join T_CFG_STAFF_MASTER sm on sh.Staff_Id = sm.Staff_Id 
where sh.AcademicYear_Id=@pAcYr and sh.Branch_ID=@pBranchId 
and sh.Semester_ID=@pSemesterId
and sh.YearOfStudy_ID=@pYearOfStudyId
and sd.Class_Section_Id=@pSectionId
and sd.Status=1 and sd.Delete_Flag=0
 and sh.Status=1 and sh.Delete_Flag=0 


end
else
begin



 SET @cols = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_ShortCode) FROM T_CFG_COURSE_MASTER CM
join T_MARK_ENTRY ME (NOLOCK) on CM.Course_Id=ME.Course_Id where ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Semester_Id=@pSemesterId and ME.Exam_Id=@pExamId and ME.AcademicYear_Id=@pAcYr FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 
 select @cols SET @query = 'Select   StuTabId,Gender, SNo as [SNO],StudentId_Rollno  as [Student Id], UniversityRegNo as  [Register No], 
 StudentName as [Student Name],Section, '+@cols+' ,Total,Rank,[Attendance Percentage]
	from (
	Select SP.Id as StuTabId,SP.Gender,SP.SNo, SP.StudentId_Rollno, SP.UniversityRegNo,SP.StudentName+'' ''+SP.Initial as StudentName,CM.Course_ShortCode, 
	(case when (ME.Is_Absent=1 and ME.Marks=0) then ''AB'' when (ME.Is_OD=1 and ME.Marks=0) then ''OD'' else ME.Marks  end) as MARKS ,
	SC.Section_Name as Section,f.Total,f.rank_No as Rank,f.[Attendance Percentage] 
	from T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	join T_CFG_SECTION_MASTER SC on SP.Section_Id=SC.id
	join fnTotalAndAttendance('+@pBranchId+','+@pYearOfStudyId+','+@pSemesterId+','+@pExamId+',
										 '''+@pfromDt+''','''+@pToDt+''','+@pAcYr+') as f on sp.Id=f.StuTabId
	where ME.AcademicYear_Id='+@pAcYr+' and ME.Branch_Id='+@pBranchId+' and ME.YearOfStudy_Id='+@pYearOfStudyId+' and 
		  ME.Semester_Id='+@pSemesterId+' and
		  ME.Exam_Id='+@pExamId+' and SP.Delete_Flag=0 and SP.Status=1 and SP.TC_ISSUED=0 
	
	) as Source pivot(MAX(MARKS) for Course_ShortCode in ('+@cols+')) as PVT  '+@pAppText+'' 

	--select * from T_CFG_STUDENT_PERSONAL_DETAIL SP where SP.Delete_Flag=0 and SP.Status=1 and SP.TC_ISSUED=0
 
PRINT @query

execute(@query)

select distinct cm.Course_Code as [Sub Code],cm.Course_Name as [Subjet Name],sm.Staff_Name as [Staff Name],SC.Section_Name as Section from T_COURSE_REGULATION_SYALLBUS_DTL rd 
join T_COURSE_REGULATION_SYALLBUS_HDR rh on rd.ID = rh.ID
join T_CFG_COURSE_MASTER cm on rd.Course_Code=cm.Course_Id
join T_STAFF_ASSIGINING_CLASS_COURSE_DTL sd on cm.Course_Id=sd.Course_Id
join T_STAFF_ASSIGINING_CLASS_COURSE_HDR sh on sd.Hdr_Id=sh.Id
left join T_CFG_STAFF_MASTER sm on sh.Staff_Id = sm.Staff_Id 
join T_CFG_SECTION_MASTER SC on sd.Class_Section_Id=SC.ID
where sh.AcademicYear_Id=@pAcYr and  sh.Branch_ID=@pBranchId 
and sh.Semester_ID=@pSemesterId
and sh.YearOfStudy_ID=@pYearOfStudyId
and sd.Status=1 and sd.Delete_Flag=0
and sh.Status=1 and sh.Delete_Flag=0 
ORDER BY SC.Section_Name ASC

end

end

--exec GetStudentMarkList 8,2,3,0,1,'2014-09-01','2014-09-03',1,3


GO
/****** Object:  StoredProcedure [dbo].[getStudentmarkPercentage]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE proc [dbo].[getStudentmarkPercentage]
as
begin
select count(a.Course_Id),ap.StudentName+' '+ap.Initial as StudentName,c.Course_Name from T_ATTENDANCEENTRY a,T_CFG_COURSE_MASTER c,T_CFG_STUDENT_PERSONAL_DETAIL ap where ap.Id=a.StudentTable_Id and a.Course_Id=c.Course_Id group by ap.StudentName,ap.Initial,c.Course_Name
end
GO
/****** Object:  StoredProcedure [dbo].[MarkNotEntered]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE proc [dbo].[MarkNotEntered]
(
 @lngExamId nvarchar(5),
 @IsRetest bit,
 @strDateExp nvarchar(50)
)
as 
begin
declare @academicYearId as nvarchar(5),@query AS NVARCHAR(MAX),@strMinimumMark nvarchar(5);
SET @academicYearId=(select id from T_CFG_ACADEMIC_YEAR where Academic_year_Status=1 and flag=0)
set @strMinimumMark=(select SettingValue from T_CFG_CONFIGURATON_SETTINGS where SettingName='PassMark');
if(@IsRetest=0)
begin

WITH MARK_NOT_ENTERED(
Academic_Year_Id,Branch_Id,YearOfStudy_Id,Semester_Id,Section_Id,Course_Id,Exam_Date,Mark_Submission_Date,ReTest_Mark_Submission_Date
)
AS
(
select DISTINCT  EXH.Academic_Year_Id,Exh.Branch_Id,Exh.YearOfStudy_Id,Exh.Semester_Id,Exh.Section_Id,EXD.Course_Id,exd.Exam_Date,exd.Mark_Submission_Date 
,EXD.ReTest_Mark_Submission_Date from T_EXAM_DEFINITION Exh
JOIN T_EXAMDEFINITION_DETAIL EXD ON Exh.Exam_Definition_Id = EXD.Exam_Definition_Id
join T_CFG_SEMESTER SEM ON Exh.Semester_Id=SEM.Semester_Id 
JOIN T_CFG_ACADEMIC_YEAR AC ON Exh.Academic_Year_Id=AC.ID
LEFT OUTER JOIN T_MARK_ENTRY ME (NOLOCK) ON EXH.Exam_Id=ME.Exam_Id AND EXH.Academic_Year_Id=ME.AcademicYear_Id 
AND Exh.Branch_Id=ME.Branch_Id AND Exh.YearOfStudy_Id=ME.YearOfStudy_Id  
AND Exh.Semester_Id = ME.Semester_Id AND Exh.Section_Id= ME.Section_Id
AND EXD.Course_Id=ME.Course_Id 
where Exh.Exam_Id=@lngExamId and Exh.Delete_Flag=0 AND EXD.Flag=0 AND
CONVERT (date, Exd.Mark_Submission_Date,105) <  CONVERT(date,cast(@strDateExp as datetime),105) and
--CONVERT (nvarchar(50), Exd.Mark_Submission_Date,105) <  CONVERT(nvarchar(50),@strDateExp,105) and
SEM.Active_Semester=1 AND SEM.Status=1 AND SEM.Delete_Flag=0
AND AC.Status=1  AND  AC.flag=0 AND AC.Academic_year_Status=1 AND ME.Mark_Entry_Id IS NULL
),
STAFF_DET(Staff_Cls_CrsDtl_Id,AcademicYear_Id,Branch_Id,YearOfStudy_Id,Semester_Id,Staff_Id,Class_Section_Id,Course_Id) 
AS
(
SELECT STFD.Id 'Staff_Cls_CrsDtl_Id', STFH.AcademicYear_Id, STFH.Branch_Id,STFH.YearOfStudy_Id,STFH.Semester_Id, STFH.Staff_Id,STFD.Class_Section_Id,STFD.Course_Id
FROM  T_STAFF_ASSIGINING_CLASS_COURSE_HDR STFH
JOIN T_STAFF_ASSIGINING_CLASS_COURSE_DTL STFD ON  STFD.Hdr_Id=STFH.Id  
JOIN T_CFG_ACADEMIC_YEAR AC ON STFH.AcademicYear_Id=AC.ID
JOIN T_CFG_SEMESTER SEM ON STFH.Semester_Id=SEM.Semester_Id 
WHERE  STFD.Delete_Flag=0 AND STFD.Status=1 AND STFH.Delete_Flag=0 AND STFH.Status=1 
AND SEM.Delete_Flag=0 AND SEM.Active_Semester=1
AND AC.flag=0 AND AC.Status=1 AND AC.Academic_year_Status=1 AND STFH.AcademicYear_Id=@academicYearId
)
SELECT B.Staff_Cls_CrsDtl_Id, A.Branch_Id as BranchId,brnch.Branch_Code as BranchName,A.YearOfStudy_Id as YearId, 
yr.YearOfStudy_Name as YearName,A.Semester_Id as SemesterId,sem.Semester_Name as SemesterName , 
sec.ID as SectionId,sec.Section_Name as SectionName,B.Course_Id as CourseId,cou.Course_Name as 
CourseName,b.Staff_Id as StaffId,stf.Staff_Name as StaffName,stf.Email_Official as StaffEmail ,CONVERT(nvarchar(50),A.Exam_Date,103) as [ExamDate], 
CONVERT(nvarchar(50),A.Mark_Submission_Date,103) as [MarkSubmissionDate], 
CONVERT(nvarchar(50),A.ReTest_Mark_Submission_Date,103) as [RetestMarkSubmission] 
FROM MARK_NOT_ENTERED A (NOLOCK)
JOIN STAFF_DET B ON A.Academic_Year_Id=B.AcademicYear_Id
join T_CFG_BRANCH brnch on a.Branch_Id=brnch.Branch_Id 
join T_CFG_YEAROFSTUDY_MASTER yr on a.YearOfStudy_Id=yr.YearOfStudy_Id
join T_CFG_SEMESTER sem on a.Semester_Id=sem.Semester_Id 
join T_CFG_SECTION_MASTER sec on a.Section_Id=sec.id 
join T_CFG_COURSE_MASTER cou on b.Course_Id=cou.Course_Id  
join T_CFG_STAFF_MASTER stf on B.Staff_Id=stf.staff_Id  
WHERE A.Branch_Id=B.Branch_Id AND A.YearOfStudy_Id= B.YearOfStudy_Id AND A.Semester_Id=B.Semester_Id AND A.Section_Id=B.Class_Section_Id 
AND A.Course_Id=B.Course_Id
ORDER BY A.Branch_Id,A.YearOfStudy_Id

end
else
begin

select distinct a.StaffClassCourseDtl_Id 'Staff_Cls_CrsDtl_Id', A.Branch_Id as BranchId,brnch.Branch_Code as BranchName,A.YearOfStudy_Id as YearId,
yr.YearOfStudy_Name as YearName,A.Semester_Id as SemesterId,sem.Semester_Name as SemesterName ,
A.Section_Id as SectionId,sec.Section_Name as SectionName,a.Course_Id as CourseId,cou.Course_Name as CourseName
,stf.Staff_Id as StaffId,stf.Staff_Name as StaffName,stf.Email_Official as StaffEmail,
CONVERT(nvarchar(50),EXD.Exam_Date,103) as [ExamDate],
CONVERT(nvarchar(50),EXD.Mark_Submission_Date,103) as [MarkSubmissionDate],
CONVERT(nvarchar(50),EXD.ReTest_Mark_Submission_Date,103) as [RetestMarkSubmission]
from T_MARK_ENTRY a (NOLOCK)
join T_CFG_BRANCH brnch on a.Branch_Id=brnch.Branch_Id
 join T_CFG_YEAROFSTUDY_MASTER yr on a.YearOfStudy_Id=yr.YearOfStudy_Id 
 join T_CFG_SEMESTER sem on a.Semester_Id=sem.Semester_Id 
 join T_CFG_SECTION_MASTER sec on a.Section_Id=sec.id 
 join T_CFG_COURSE_MASTER cou on a.Course_Id=cou.Course_Id
 join T_STAFF_ASSIGINING_CLASS_COURSE_DTL STFD on STFD.Id = a.StaffClassCourseDtl_Id
 join T_STAFF_ASSIGINING_CLASS_COURSE_HDR STFH on STFH.Id= STFD.Hdr_Id 
 join T_CFG_STAFF_MASTER stf on STFH.Staff_Id=stf.staff_Id 
 join T_EXAMDEFINITION_DETAIL EXD on a.Course_Id = EXD.Course_Id
 join T_EXAM_DEFINITION Exh on EXD.Exam_Definition_Id = Exh.Exam_Definition_Id
 join T_CFG_ACADEMIC_YEAR AC on a.AcademicYear_Id = AC.ID
 join T_CFG_STUDENT_PERSONAL_DETAIL STU on a.StudentTable_Id = STU.Id 
 where a.Branch_Id=exh.Branch_Id and a.YearOfStudy_Id=exh.YearOfStudy_Id and a.Semester_Id=exh.Semester_Id 
and a.Section_Id=exh.Section_Id 
and exh.Exam_Id=a.Exam_Id
and a.AcademicYear_Id =@academicYearId
and exh.Academic_Year_Id=a.AcademicYear_Id
and CONVERT (date, Exd.ReTest_Mark_Submission_Date,105) <  CONVERT(date,cast(@strDateExp as datetime),105) 
--and CONVERT (nvarchar(50), Exd.ReTest_Mark_Submission_Date,103)  < CONVERT(nvarchar(50),@strDateExp,103)
and marks< @strMinimumMark   and RetestMarks is null and a.exam_id= @lngExamId   and sem.Active_Semester=1 
and AC.flag=0 and AC.Status=1 and AC.Academic_year_Status=1 
and stu.Delete_Flag=0 and stu.TC_ISSUED=0 and stu.status=1 AND STFH.AcademicYear_Id=@academicYearId
and exh.Delete_Flag=0 and  EXD.flag=0
order by a.Branch_Id,a.YearOfStudy_Id,a.Section_Id

end


end


--exec Marknotentered 1,0,'2015-09-27'






GO
/****** Object:  StoredProcedure [dbo].[newTest]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


CREATE Procedure [dbo].[newTest]
@pInsitutionId bigint, @pBranchId bigint, @pYearofStudyId bigint,@pAcademicYear bigint,@pFromDate nvarchar(20),@pToDate nvarchar(20)
As Begin
;With StuInfo(StuTabId,StuName, StuUnivRegNo,StuBatchId,SemId)
As
(
select id,StudentName +' '+ Initial as StudentName,UniversityRegNo,AcademicBatch_Id,Semester_Id
from T_CFG_STUDENT_PERSONAL_DETAIL
where AcademicYear_Id=@pAcademicYear and Institution_Id=@pInsitutionId and Branch_Id=@pBranchId and YearOfStudy_Id=@pYearofStudyId 
)
,
regulationInfo(regId)
As
(
select Regulation_Id from T_CFG_ACADEMICBATCH d where d.Id=(select Top 1 StuBatchId from StuInfo)
)
,

CourseInfo(SubId,SubCode,SubName)
As
(
select c.Course_Id,c.Course_Code,c.Course_Name from T_COURSE_REGULATION_SYALLBUS_HDR a
join T_COURSE_REGULATION_SYALLBUS_DTL b on a.ID = b.ID
join T_CFG_COURSE_MASTER c on b.course_code=c.Course_Id
where a.Regulation_ID=(select TOP 1 regId from regulationInfo)  and
	  a.Institution_ID=@pInsitutionId and a.Branch_Id=@pBranchId and a.YearOfStudy_Id=@pYearofStudyId and a.Semester_ID=(select top 1 SemId from StuInfo) and c.Course_Code not in ('','lab','-','PR','P and','Library')

	  
)
,

pivotInfo ([Registration_Number],StuName,SubId, [CourseName],Present,[Absent])
As
(

select StuUnivRegNo as [Registration Number], StuName as [StudentName], SubId, SubName as [CourseName],isnull(P,0) as Present,isnull(Ab,0) as [Absent]
from
(
select stu.StuUnivRegNo, stu.StuName,CourseInfo.SubId,CourseInfo.SubName,atndnce.Attendance_Status,count(atndnce.Attendance_Status) as AttendanceStatus from T_ATTENDANCEENTRY atndnce 
join StuInfo stu on atndnce.StudentTable_Id= stu.StuTabId
join CourseInfo on atndnce.Course_Id=CourseInfo.SubId
where atndnce.attendance_date between CONVERT(varchar(50),@pFromDate,103) And CONVERT(varchar(50),@pToDate,103) group by stu.StuName,CourseInfo.SubId,CourseInfo.SubName,atndnce.Attendance_Status,stu.StuUnivRegNo -- order by atndnce.Course_Id,atndnce.StudentTable_Id
) as source
pivot (max(AttendanceStatus) for Attendance_Status in([P],[Ab])) as pvt

)

select [Registration_Number], [StuName] as StudentName,SubId as CourseId,CourseName,Present as Attended, [Absent],
	   (Present+[Absent])as Conducted,
	   cast((Present*100)/((Present+[Absent])) as decimal(10,2))  as [Attendance_Percentage]
 from pivotInfo   order by subid

end
GO
/****** Object:  StoredProcedure [dbo].[pro_getAllRolesandpermssion]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO



CREATE procedure [dbo].[pro_getAllRolesandpermssion]
@RoleId int
as
begin
 select m.Id as ModeleId, m.ModuleName,sm.Id as SubModuleId,sm.SubModuleName,s.Id as ScreenId,
 s.ScreenName 
 from T_CFG_MODULEPERMISSION mp,T_CFG_MODULETOSCREENS ms,T_CFG_MODULE m,T_CFG_SUBMODULE sm,T_CFG_SCREEN s
 where mp.ModuleId=m.Id and mp.ModuleId=ms.ModuleId and ms.SubModuleId=sm.Id
    and ms.ScreenId=s.Id
    and mp.RoleId=@RoleId
    end 
    
    
    
  

     


GO
/****** Object:  StoredProcedure [dbo].[pro_getAllRolesandpermssionForUserAccess]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO



CREATE procedure [dbo].[pro_getAllRolesandpermssionForUserAccess]
@RoleIds varchar(500)
as
begin
 select distinct m.Id as ModeleId, m.ModuleName,sm.Id as SubModuleId,sm.SubModuleName,s.Id as ScreenId,
 s.ScreenName 
 from T_CFG_MODULEPERMISSION mp,T_CFG_MODULETOSCREENS ms,T_CFG_MODULE m,T_CFG_SUBMODULE sm,T_CFG_SCREEN s
 where mp.ModuleId=m.Id and mp.ModuleId=ms.ModuleId and ms.SubModuleId=sm.Id
    and ms.ScreenId=s.Id
    and mp.RoleId in(Select s from SplitString(@RoleIds,','))   
    end 


GO
/****** Object:  StoredProcedure [dbo].[Pro_GetStudentDetails]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE procedure [dbo].[Pro_GetStudentDetails]
(
@plstBranchId NVARCHAR(100),
@plstYearId NVARCHAR(100),
@plstSemesterId NVARCHAR(100),
@plstSectionId NVARCHAR(100),
@pAcademicYearId BIGINT,
@pOrderByName nvarchar(50)
)
AS 
BEGIN   

if(@pOrderByName='Register No')
Begin

SELECT BR.Branch_Code AS [Branch], STU.SNo AS [Student S.No.], STU.StudentName+' '+STU.Initial as StudentName, STU.StudentId_Rollno AS [Student Id],
STU.UniversityRegNo, Y.YearOfStudy_Name AS [Year of Study], SEM.Semester_Name AS [Semester], SEC.Section_Name AS [Section], 
PRO.Programme_Name AS [Programme], STU.Gender, CONVERT(NVARCHAR(50), STU.dateofbirth, 103) AS [DOB], STU.MobileNo AS [Student Mobile], 
STU.Emailid AS [E-Mail], P.F_Name AS [FatherName], P.M_Name AS [MotherName], P.F_Mobile_Number AS [Father Mobile], 
CASE STU.Is_Hostel WHEN 1 THEN 'Yes' ELSE 'No' END AS Hostel,INS.Institution_Logo as Institution_Logo
FROM T_CFG_STUDENT_PERSONAL_DETAIL STU
JOIN T_CFG_STUDENT_PARENT_DETAIL P ON STU.Id = P.StudentTable_Id
JOIN T_CFG_PROGRAMME PRO ON STU.Programme_Id = PRO.Programme_Id
JOIN T_CFG_BRANCH BR ON STU.Branch_Id = BR.Branch_Id
JOIN T_CFG_YEAROFSTUDY_MASTER Y ON STU.YearOfStudy_Id = Y.YearOfStudy_Id
JOIN T_CFG_SEMESTER SEM ON STU.Semester_Id = SEM.Semester_Id
JOIN T_CFG_SECTION_MASTER SEC ON STU.Section_Id = SEC.ID
JOIN T_CFG_INSTITUTION INS ON STU.Institution_Id= INS.Institution_Id
WHERE STU.Status = 1 AND STU.Delete_Flag = 0 AND STU.TC_ISSUED = 0
AND stu.Branch_Id IN (SELECT id FROM CSVToTable(@plstBranchId)) AND stu.YearOfStudy_Id IN (SELECT id FROM CSVToTable(@plstYearId)) AND 
stu.Semester_Id IN (SELECT id FROM CSVToTable(@plstSemesterId)) AND stu.Section_Id IN (SELECT id FROM CSVToTable(@plstSectionId))
AND STU.AcademicYear_Id = @pAcademicYearId
ORDER BY BR.Branch_Code , Y.YearOfStudy_Name, SEM.Semester_Name, SEC.Section_Name,STU.UniversityRegNo
end

else if(@pOrderByName='Student Id')
Begin
SELECT BR.Branch_Code AS [Branch], STU.SNo AS [Student S.No.], STU.StudentName+' '+STU.Initial as StudentName, STU.StudentId_Rollno AS [Student Id],
STU.UniversityRegNo, Y.YearOfStudy_Name AS [Year of Study], SEM.Semester_Name AS [Semester], SEC.Section_Name AS [Section], 
PRO.Programme_Name AS [Programme], STU.Gender, CONVERT(NVARCHAR(50), STU.dateofbirth, 103) AS [DOB], STU.MobileNo AS [Student Mobile], 
STU.Emailid AS [E-Mail], P.F_Name AS [FatherName], P.M_Name AS [MotherName], P.F_Mobile_Number AS [Father Mobile], 
CASE STU.Is_Hostel WHEN 1 THEN 'Yes' ELSE 'No' END AS Hostel,INS.Institution_Logo as Institution_Logo
FROM T_CFG_STUDENT_PERSONAL_DETAIL STU
JOIN T_CFG_STUDENT_PARENT_DETAIL P ON STU.Id = P.StudentTable_Id
JOIN T_CFG_PROGRAMME PRO ON STU.Programme_Id = PRO.Programme_Id
JOIN T_CFG_BRANCH BR ON STU.Branch_Id = BR.Branch_Id
JOIN T_CFG_YEAROFSTUDY_MASTER Y ON STU.YearOfStudy_Id = Y.YearOfStudy_Id
JOIN T_CFG_SEMESTER SEM ON STU.Semester_Id = SEM.Semester_Id
JOIN T_CFG_SECTION_MASTER SEC ON STU.Section_Id = SEC.ID
JOIN T_CFG_INSTITUTION INS ON STU.Institution_Id= INS.Institution_Id
WHERE STU.Status = 1 AND STU.Delete_Flag = 0 AND STU.TC_ISSUED = 0
AND stu.Branch_Id IN (SELECT id FROM CSVToTable(@plstBranchId)) AND stu.YearOfStudy_Id IN (SELECT id FROM CSVToTable(@plstYearId)) AND 
stu.Semester_Id IN (SELECT id FROM CSVToTable(@plstSemesterId)) AND stu.Section_Id IN (SELECT id FROM CSVToTable(@plstSectionId))
AND STU.AcademicYear_Id = @pAcademicYearId
ORDER BY BR.Branch_Code , Y.YearOfStudy_Name, SEM.Semester_Name, SEC.Section_Name, STU.StudentId_Rollno
end

else if(@pOrderByName='Student Name')
Begin
SELECT BR.Branch_Code AS [Branch], STU.SNo AS [Student S.No.], STU.StudentName+' '+STU.Initial as StudentName, STU.StudentId_Rollno AS [Student Id],
STU.UniversityRegNo, Y.YearOfStudy_Name AS [Year of Study], SEM.Semester_Name AS [Semester], SEC.Section_Name AS [Section], 
PRO.Programme_Name AS [Programme], STU.Gender, CONVERT(NVARCHAR(50), STU.dateofbirth, 103) AS [DOB], STU.MobileNo AS [Student Mobile], 
STU.Emailid AS [E-Mail], P.F_Name AS [FatherName], P.M_Name AS [MotherName], P.F_Mobile_Number AS [Father Mobile], 
CASE STU.Is_Hostel WHEN 1 THEN 'Yes' ELSE 'No' END AS Hostel,INS.Institution_Logo as Institution_Logo
FROM T_CFG_STUDENT_PERSONAL_DETAIL STU
JOIN T_CFG_STUDENT_PARENT_DETAIL P ON STU.Id = P.StudentTable_Id
JOIN T_CFG_PROGRAMME PRO ON STU.Programme_Id = PRO.Programme_Id
JOIN T_CFG_BRANCH BR ON STU.Branch_Id = BR.Branch_Id
JOIN T_CFG_YEAROFSTUDY_MASTER Y ON STU.YearOfStudy_Id = Y.YearOfStudy_Id
JOIN T_CFG_SEMESTER SEM ON STU.Semester_Id = SEM.Semester_Id
JOIN T_CFG_SECTION_MASTER SEC ON STU.Section_Id = SEC.ID
JOIN T_CFG_INSTITUTION INS ON STU.Institution_Id= INS.Institution_Id
WHERE STU.Status = 1 AND STU.Delete_Flag = 0 AND STU.TC_ISSUED = 0
AND stu.Branch_Id IN (SELECT id FROM CSVToTable(@plstBranchId)) AND stu.YearOfStudy_Id IN (SELECT id FROM CSVToTable(@plstYearId)) AND 
stu.Semester_Id IN (SELECT id FROM CSVToTable(@plstSemesterId)) AND stu.Section_Id IN (SELECT id FROM CSVToTable(@plstSectionId))
AND STU.AcademicYear_Id = @pAcademicYearId
ORDER BY BR.Branch_Code , Y.YearOfStudy_Name, SEM.Semester_Name, SEC.Section_Name, STU.StudentName
end

else if(@pOrderByName='SNO')
Begin
SELECT BR.Branch_Code AS [Branch], STU.SNo AS [Student S.No.], STU.StudentName+' '+STU.Initial as StudentName, STU.StudentId_Rollno AS [Student Id],
STU.UniversityRegNo, Y.YearOfStudy_Name AS [Year of Study], SEM.Semester_Name AS [Semester], SEC.Section_Name AS [Section], 
PRO.Programme_Name AS [Programme], STU.Gender, CONVERT(NVARCHAR(50), STU.dateofbirth, 103) AS [DOB], STU.MobileNo AS [Student Mobile], 
STU.Emailid AS [E-Mail], P.F_Name AS [FatherName], P.M_Name AS [MotherName], P.F_Mobile_Number AS [Father Mobile], 
CASE STU.Is_Hostel WHEN 1 THEN 'Yes' ELSE 'No' END AS Hostel,INS.Institution_Logo as Institution_Logo
FROM T_CFG_STUDENT_PERSONAL_DETAIL STU
JOIN T_CFG_STUDENT_PARENT_DETAIL P ON STU.Id = P.StudentTable_Id
JOIN T_CFG_PROGRAMME PRO ON STU.Programme_Id = PRO.Programme_Id
JOIN T_CFG_BRANCH BR ON STU.Branch_Id = BR.Branch_Id
JOIN T_CFG_YEAROFSTUDY_MASTER Y ON STU.YearOfStudy_Id = Y.YearOfStudy_Id
JOIN T_CFG_SEMESTER SEM ON STU.Semester_Id = SEM.Semester_Id
JOIN T_CFG_SECTION_MASTER SEC ON STU.Section_Id = SEC.ID
JOIN T_CFG_INSTITUTION INS ON STU.Institution_Id= INS.Institution_Id
WHERE STU.Status = 1 AND STU.Delete_Flag = 0 AND STU.TC_ISSUED = 0
AND stu.Branch_Id IN (SELECT id FROM CSVToTable(@plstBranchId)) AND stu.YearOfStudy_Id IN (SELECT id FROM CSVToTable(@plstYearId)) AND 
stu.Semester_Id IN (SELECT id FROM CSVToTable(@plstSemesterId)) AND stu.Section_Id IN (SELECT id FROM CSVToTable(@plstSectionId))
AND STU.AcademicYear_Id = @pAcademicYearId
ORDER BY BR.Branch_Code , Y.YearOfStudy_Name, SEM.Semester_Name, SEC.Section_Name, STU.SNo
end

else if(@pOrderByName='Male')
Begin
SELECT BR.Branch_Code AS [Branch], STU.SNo AS [Student S.No.], STU.StudentName+' '+STU.Initial as StudentName, STU.StudentId_Rollno AS [Student Id],
STU.UniversityRegNo, Y.YearOfStudy_Name AS [Year of Study], SEM.Semester_Name AS [Semester], SEC.Section_Name AS [Section], 
PRO.Programme_Name AS [Programme], STU.Gender, CONVERT(NVARCHAR(50), STU.dateofbirth, 103) AS [DOB], STU.MobileNo AS [Student Mobile], 
STU.Emailid AS [E-Mail], P.F_Name AS [FatherName], P.M_Name AS [MotherName], P.F_Mobile_Number AS [Father Mobile], 
CASE STU.Is_Hostel WHEN 1 THEN 'Yes' ELSE 'No' END AS Hostel,INS.Institution_Logo as Institution_Logo
FROM T_CFG_STUDENT_PERSONAL_DETAIL STU
JOIN T_CFG_STUDENT_PARENT_DETAIL P ON STU.Id = P.StudentTable_Id
JOIN T_CFG_PROGRAMME PRO ON STU.Programme_Id = PRO.Programme_Id
JOIN T_CFG_BRANCH BR ON STU.Branch_Id = BR.Branch_Id
JOIN T_CFG_YEAROFSTUDY_MASTER Y ON STU.YearOfStudy_Id = Y.YearOfStudy_Id
JOIN T_CFG_SEMESTER SEM ON STU.Semester_Id = SEM.Semester_Id
JOIN T_CFG_SECTION_MASTER SEC ON STU.Section_Id = SEC.ID
JOIN T_CFG_INSTITUTION INS ON STU.Institution_Id= INS.Institution_Id
WHERE STU.Status = 1 AND STU.Delete_Flag = 0 AND STU.TC_ISSUED = 0
AND stu.Branch_Id IN (SELECT id FROM CSVToTable(@plstBranchId)) AND stu.YearOfStudy_Id IN (SELECT id FROM CSVToTable(@plstYearId)) AND 
stu.Semester_Id IN (SELECT id FROM CSVToTable(@plstSemesterId)) AND stu.Section_Id IN (SELECT id FROM CSVToTable(@plstSectionId))
AND STU.AcademicYear_Id = @pAcademicYearId
ORDER BY BR.Branch_Code , Y.YearOfStudy_Name, SEM.Semester_Name, SEC.Section_Name, STU.Gender DESC
end

else if(@pOrderByName='Female')
Begin
SELECT BR.Branch_Code AS [Branch], STU.SNo AS [Student S.No.], STU.StudentName+' '+STU.Initial as StudentName, STU.StudentId_Rollno AS [Student Id],
STU.UniversityRegNo, Y.YearOfStudy_Name AS [Year of Study], SEM.Semester_Name AS [Semester], SEC.Section_Name AS [Section], 
PRO.Programme_Name AS [Programme], STU.Gender, CONVERT(NVARCHAR(50), STU.dateofbirth, 103) AS [DOB], STU.MobileNo AS [Student Mobile], 
STU.Emailid AS [E-Mail], P.F_Name AS [FatherName], P.M_Name AS [MotherName], P.F_Mobile_Number AS [Father Mobile], 
CASE STU.Is_Hostel WHEN 1 THEN 'Yes' ELSE 'No' END AS Hostel,INS.Institution_Logo as Institution_Logo
FROM T_CFG_STUDENT_PERSONAL_DETAIL STU
JOIN T_CFG_STUDENT_PARENT_DETAIL P ON STU.Id = P.StudentTable_Id
JOIN T_CFG_PROGRAMME PRO ON STU.Programme_Id = PRO.Programme_Id
JOIN T_CFG_BRANCH BR ON STU.Branch_Id = BR.Branch_Id
JOIN T_CFG_YEAROFSTUDY_MASTER Y ON STU.YearOfStudy_Id = Y.YearOfStudy_Id
JOIN T_CFG_SEMESTER SEM ON STU.Semester_Id = SEM.Semester_Id
JOIN T_CFG_SECTION_MASTER SEC ON STU.Section_Id = SEC.ID
JOIN T_CFG_INSTITUTION INS ON STU.Institution_Id= INS.Institution_Id
WHERE STU.Status = 1 AND STU.Delete_Flag = 0 AND STU.TC_ISSUED = 0
AND stu.Branch_Id IN (SELECT id FROM CSVToTable(@plstBranchId)) AND stu.YearOfStudy_Id IN (SELECT id FROM CSVToTable(@plstYearId)) AND 
stu.Semester_Id IN (SELECT id FROM CSVToTable(@plstSemesterId)) AND stu.Section_Id IN (SELECT id FROM CSVToTable(@plstSectionId))
AND STU.AcademicYear_Id = @pAcademicYearId
ORDER BY BR.Branch_Code , Y.YearOfStudy_Name, SEM.Semester_Name, SEC.Section_Name, STU.Gender
end

END

--EXEC Pro_GetStudentDetails '1', '2', '4', '1', '3'


--EXEC Pro_GetStudentDetails '1,2', '2,3', '4,6', '1,2', '3','Male'



GO
/****** Object:  StoredProcedure [dbo].[pro_getUserRoleForUpdate]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[pro_getUserRoleForUpdate]
@UserId int
as
begin
 select r.id as Roleid,r.rolename from T_CFG_ROLES r
 where r.id not in(select Role_Id from T_CFG_USER_ROLE where User_Id = @UserId and Delete_Flag=0)
 end
GO
/****** Object:  StoredProcedure [dbo].[Proc_College_Strength]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[Proc_College_Strength]
(
@plstBranchID nvarchar(100),
@plstYearId nvarchar(100),
@pAcademicYr bigint
)
as
select 
br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name
,count(case when stu.Gender like 'M%' Then 1 end) as Boys
,count(case when stu.Gender like 'F%' Then 1 end) as Girls
,count(stu.Id) as [Total Strength]
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_CFG_BRANCH br on stu.Branch_Id=br.Branch_Id
join T_CFG_YEAROFSTUDY_MASTER Yr on stu.YearOfStudy_Id = Yr.YearOfStudy_Id
join T_CFG_SEMESTER sem on stu.Semester_Id=sem.Semester_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id= sec.ID
where stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0 and stu.AcademicYear_Id=@pAcademicYr
and stu.Branch_Id in (select id from CSVToTable(@plstBranchID))and stu.YearOfStudy_Id in (select id from CSVToTable(@plstYearId))
group by br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name
order by br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name



--exec Proc_College_Strength '1,2,8','1,2,3',3




GO
/****** Object:  StoredProcedure [dbo].[Proc_College_Strength_SecWise]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE Proc [dbo].[Proc_College_Strength_SecWise]
(
@plstBranchID nvarchar(100),
@plstYearId nvarchar(100),
@pAcademicYr bigint
)
as

select 
br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name,sec.Section_Name
,count(case when stu.Gender like 'M%' Then 1 end) as Boys
,count(case when stu.Gender like 'F%' Then 1 end) as Girls
,count(stu.Id) as [Total Strength]
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_CFG_BRANCH br on stu.Branch_Id=br.Branch_Id
join T_CFG_YEAROFSTUDY_MASTER Yr on stu.YearOfStudy_Id = Yr.YearOfStudy_Id
join T_CFG_SEMESTER sem on stu.Semester_Id=sem.Semester_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id= sec.ID
where stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0 and stu.AcademicYear_Id=@pAcademicYr
and stu.Branch_Id in (select id from CSVToTable(@plstBranchID))and stu.YearOfStudy_Id in (select id from CSVToTable(@plstYearId))
group by br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name,sec.Section_Name
order by br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name,sec.Section_Name






--exec Proc_College_Strength_SecWise '1,2,8','1,2,3',3




GO
/****** Object:  StoredProcedure [dbo].[Proc_Interview_Count]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[Proc_Interview_Count]

@pAcademicYr BIGINT

AS
BEGIN
with Interview(Student_Id,[Eligible Count],[Not Attended Count])
as
(
select  Istu.Student_Id,count(Istu.Student_Id) 'Eligible Count'
,count(case when Istu.InterviewStatus_Id=1 then 1 else NULL end) 'Not Attended Count'
from T_RECRUITER rec
join T_INTERVIEW_STUDENTS Istu on rec.id=Istu.Recruiter_Id
where rec.AcademicYear_Id=@pAcademicYr
group by Istu.Student_Id
having count(case when Istu.InterviewStatus_Id=1 then 1 else NULL end) >0
)
,
StuInfo(Id,StudentId_Rollno,UniversityRegNo,Branch_Code,StudentName)
as
(

select Id,StudentId_Rollno,UniversityRegNo,brnch.Branch_Code,StudentName 
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_CFG_BRANCH brnch on stu.Branch_Id=brnch.Branch_Id
join Interview I on stu.Id=I.Student_Id
)
select b.StudentId_Rollno 'StudentId'
,b.UniversityRegNo 'RegisterNo'
,b.Branch_Code 'Branch'
,b.StudentName 'StudentName'
,a.[Eligible Count] 'EligibleCount'
,a.[Not Attended Count] 'NotAttendedCount'
from Interview a
join StuInfo b on a.Student_Id=b.Id


END

--exec proc_interview_count 4

GO
/****** Object:  StoredProcedure [dbo].[Proc_Placement]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[Proc_Placement]
as begin
select ac.Academic_year_code 'Academic Year'
,count(case when (rec.Delete_Flag=0 and I.Delete_Flag=0) then 1 else NULL end) 'EligibleStudents'
,count(case when (rec.Delete_Flag=0 and I.Delete_Flag=0 and I.IsAppeared=1) then 1 else NULL end) 'StudentsAppeared'
,count(case when (rec.Delete_Flag=0 and I.Delete_Flag=0 and I.InterviewStatus_Id=5) then 1 else NULL end) 'SelectedStudents'
 from T_RECRUITER rec
 right outer join T_CFG_ACADEMIC_YEAR ac on rec.AcademicYear_Id=ac.ID
 left outer join T_INTERVIEW_STUDENTS I on rec.Id=I.Recruiter_Id
group by rec.AcademicYear_Id,ac.Academic_year_code
end

--exec Proc_Placement
GO
/****** Object:  StoredProcedure [dbo].[Proc_Strength_of_students]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


CREATE procedure [dbo].[Proc_Strength_of_students]
(
@branch_id nvarchar(100),
@year_id nvarchar(100),
--@sem_id nvarchar(100),
--@sec_id nvarchar(100),
@academic_yr_id bigint,
@Is_SecWise bit
)
as begin

declare @tBranchId table
(
	tBranchId bigint
)

insert into @tBranchId (tBranchId) 
select s from SplitString(@branch_id,',')

declare @tYrId table
(
	tYrId bigint
)

insert into @tYrId (tYrId)
select s from SplitString(@year_id,',')


if(@Is_SecWise=1)
begin
select br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name,sec.Section_Name,count(stu.id) Strength from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_CFG_BRANCH Br on stu.Branch_Id=br.Branch_Id
join T_CFG_YEAROFSTUDY_MASTER Yr on stu.YearOfStudy_Id=yr.YearOfStudy_Id
join T_CFG_SEMESTER sem on stu.Semester_Id = sem.Semester_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
where stu.Status=1 and stu.Delete_Flag=0 and stu.TC_ISSUED=0
and stu.Branch_Id in (select tBranchId from @tBranchId ) and stu.YearOfStudy_Id in (select tYrId from @tYrId) --and stu.Semester_Id in (@sem_id) and stu.Section_Id in (@sec_id)
and stu.AcademicYear_Id=@academic_yr_id
group by br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name,sec.Section_Name
order by br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name,sec.Section_Name
end

else
begin
select br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name,count(stu.id) Strength from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_CFG_BRANCH Br on stu.Branch_Id=br.Branch_Id
join T_CFG_YEAROFSTUDY_MASTER Yr on stu.YearOfStudy_Id=yr.YearOfStudy_Id
join T_CFG_SEMESTER sem on stu.Semester_Id = sem.Semester_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
where stu.Status=1 and stu.Delete_Flag=0 and stu.TC_ISSUED=0
and stu.Branch_Id in (select tBranchId from @tBranchId) and stu.YearOfStudy_Id in (select tYrId from @tYrId) --and stu.Semester_Id in (@sem_id) and stu.Section_Id in (@sec_id)
and stu.AcademicYear_Id=@academic_yr_id
group by br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name
order by br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name
end
end



--select * from SplitString('1,2,3,4',',')

GO
/****** Object:  StoredProcedure [dbo].[sp_AttendanceReportLetterPad]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[sp_AttendanceReportLetterPad]
@StudentTableId NVARCHAR(50),
@FromDate DateTime,
@ToDate DateTime
AS
BEGIN

SELECT [Attendance Date], CASE WHEN [No Of Period] = 7 THEN 'FULL DAY' ELSE [No Of Period] END AS [No Of Period]    FROM
(SELECT CONVERT(NVARCHAR(50), Attendance_Date, 103) AS [Attendance Date], CONVERT(NVARCHAR(15), COUNT(CASE A.Attendance_Status WHEN 'AB' THEN '1' END)) AS [No Of Period]
		FROM T_ATTENDANCEENTRY A (NOLOCK)
		JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON A.StudentTable_Id = S.Id
		WHERE A.StudentTable_Id = @StudentTableId
		AND A.Attendance_Date between @FromDate AND @ToDate 
		AND S.Status=1 AND S.Delete_Flag=0 AND S.TC_ISSUED=0
		GROUP BY Attendance_Date, Attendance_Status
		HAVING COUNT(CASE A.Attendance_Status WHEN 'AB' THEN '1' END) > 0) AS TEMP

SELECT S.StudentName+' '+S.Initial as StudentName, S.SNo, B.Branch_Code, SEC.Section_Name, Y.YearOfStudy_Name, P.Programme_Name, SEM.Semester_Name, S.FatherName,
	   S.PermenentAddress, S.City, S.Pincode	 FROM T_CFG_STUDENT_PERSONAL_DETAIL S
		 JOIN T_CFG_BRANCH B ON S.Branch_Id = B.Branch_Id
		 JOIN T_CFG_YEAROFSTUDY_MASTER Y ON S.YearOfStudy_Id = Y.YearOfStudy_Id
		 JOIN T_CFG_SEMESTER SEM ON S.Semester_Id = SEM.Semester_Id
		 JOIN T_CFG_SECTION_MASTER SEC ON S.Section_Id = SEC.ID
		 JOIN T_CFG_PROGRAMME P ON B.Programme_Id = P.Programme_Id
		 WHERE S.Id = @StudentTableId AND S.Status=1 AND S.Delete_Flag=0 AND S.TC_ISSUED=0

END
--EXEC sp_AttendanceReportLetterPad 1655,'2014-01-01', '2014-08-12'
GO
/****** Object:  StoredProcedure [dbo].[Sp_BranchWisePoor]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO




CREATE procedure [dbo].[Sp_BranchWisePoor]
@pInsitutionId nvarchar(50), 
@pBranchId nvarchar(50), 
@pYearofStudyId nvarchar(50),
@pSemesterId nvarchar(50),
@pExamId nvarchar(50),
@pNoofToppers nvarchar(50),
@pFlagAddLab bit,
@pIsOverAllRpt bit,
@pAcYr nvarchar(10)
As Begin
Declare @pRegulation bigint,@pAcademicYear bigint, @cols1 AS NVARCHAR(MAX), @query1 AS NVARCHAR(MAX); 
SET @pAcademicYear=(select ID from T_CFG_ACADEMIC_YEAR Where Academic_year_Status = 1);



SET @pRegulation=(select Regulation_Id from T_CFG_ACADEMICBATCH d
				 where d.Id=(select Top 1 AcademicBatch_Id from T_CFG_STUDENT_PERSONAL_DETAIL
						where AcademicYear_Id=@pAcademicYear and Institution_Id=@pInsitutionId and Branch_Id=@pBranchId and YearOfStudy_Id=@pYearofStudyId
						and status=1 and Delete_Flag=0 and TC_ISSUED=0));

if(@pFlagAddLab=0)
begin
SET @cols1 = STUFF((SELECT distinct ','+ QUOTENAME( c.Course_Code) from T_COURSE_REGULATION_SYALLBUS_HDR a
join T_COURSE_REGULATION_SYALLBUS_DTL b on  a.ID=b.ID
join T_CFG_COURSE_MASTER c on b.Course_Code = c.Course_Id
where a.Regulation_ID=@pRegulation and a.Institution_ID=@pInsitutionId and a.Branch_Id=@pBranchId and a.YearOfStudy_Id=@pYearofStudyId and a.Semester_Id=@pSemesterId 
and c.TutorialType!='Practical' and c.Course_Code not in ('','lab','-','PR','P and','Library') and a.Flag=0 and b.FLAG=0
FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 
end
else
begin
SET @cols1 = STUFF((SELECT distinct ','+ QUOTENAME( c.Course_Code) from T_COURSE_REGULATION_SYALLBUS_HDR a
join T_COURSE_REGULATION_SYALLBUS_DTL b on  a.ID=b.ID
join T_CFG_COURSE_MASTER c on b.Course_Code = c.Course_Id
where a.Regulation_ID=@pRegulation and a.Institution_ID=@pInsitutionId and a.Branch_Id=@pBranchId and a.YearOfStudy_Id=@pYearofStudyId and a.Semester_Id=@pSemesterId 
 and c.Course_Code not in ('','lab','-','PR','P and','Library') and a.Flag=0 and b.FLAG=0
FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 
end


if(@pIsOverAllRpt=0)  ---get the section wise report.
begin
SET @query1 = 'select BranchName,[Student Id], UniversityRegNo,StudentName,Section_Name,'+@cols1+',Total
from
(

select * from fnPoorWithSection('+@pInsitutionId+','+@pBranchId+','+@pYearofStudyId+','+@pSemesterId+','+@pExamId+','+@pNoofToppers+','+@pAcYr+')
) as source
pivot(max(marks) for Course_Code in ('+@cols1+')) as pvt order by Section_Name,rank_No'
end
else
begin
SET @query1 = 'select BranchName,[Student Id], UniversityRegNo,StudentName,Section_Name,'+@cols1+',Total
from
(

select * from fnPoorOverAll('+@pInsitutionId+','+@pBranchId+','+@pYearofStudyId+','+@pSemesterId+','+@pExamId+','+@pNoofToppers+','+@pAcYr+')
) as source
pivot(max(marks) for Course_Code in ('+@cols1+')) as pvt order by rank_No'
end


PRINT @query1

execute(@query1)

end



--select * from fnToppersWithSection('+@pInsitutionId+','+@pBranchId+','+@pYearofStudyId+','+@pSemesterId+','+@pExamId+')


--select * from fnToppersWithSection(1,0,2,3,1,3)
--exec Sp_BranchWiseToppers 1,1,2,3,1,3,0,0

--select * from T_CFG_COURSE_MASTER where course_code in('cs6301','cs6302','cs6303','cs6304','cs6311','cs6312')



GO
/****** Object:  StoredProcedure [dbo].[Sp_BranchWiseToppers]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO




CREATE procedure [dbo].[Sp_BranchWiseToppers]
@pInsitutionId nvarchar(50), 
@pBranchId nvarchar(50), 
@pYearofStudyId nvarchar(50),
@pSemesterId nvarchar(50),
@pExamId nvarchar(50),
@pNoofToppers nvarchar(50),
@pFlagAddLab bit,
@pIsOverAllRpt bit,
@pAcYr nvarchar(10)
As Begin
Declare @pRegulation bigint,@pAcademicYear bigint, @cols1 AS NVARCHAR(MAX), @query1 AS NVARCHAR(MAX); 
SET @pAcademicYear=(select ID from T_CFG_ACADEMIC_YEAR Where Academic_year_Status = 1);



SET @pRegulation=(select Regulation_Id from T_CFG_ACADEMICBATCH d
				 where d.Id=(select Top 1 AcademicBatch_Id from T_CFG_STUDENT_PERSONAL_DETAIL
						where AcademicYear_Id=@pAcademicYear and Institution_Id=@pInsitutionId and Branch_Id=@pBranchId and YearOfStudy_Id=@pYearofStudyId
						and status=1 and Delete_Flag=0 and TC_ISSUED=0));

if(@pFlagAddLab=0)
begin
SET @cols1 = STUFF((SELECT distinct ','+ QUOTENAME( c.Course_Code) from T_COURSE_REGULATION_SYALLBUS_HDR a
join T_COURSE_REGULATION_SYALLBUS_DTL b on  a.ID=b.ID
join T_CFG_COURSE_MASTER c on b.Course_Code = c.Course_Id
where a.Regulation_ID=@pRegulation and a.Institution_ID=@pInsitutionId and a.Branch_Id=@pBranchId and a.YearOfStudy_Id=@pYearofStudyId and a.Semester_Id=@pSemesterId 
and c.TutorialType!='Practical' and c.Course_Code not in ('','lab','-','PR','P and','Library') and a.Flag=0 and b.FLAG=0
FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 
end
else
begin
SET @cols1 = STUFF((SELECT distinct ','+ QUOTENAME( c.Course_Code) from T_COURSE_REGULATION_SYALLBUS_HDR a
join T_COURSE_REGULATION_SYALLBUS_DTL b on  a.ID=b.ID
join T_CFG_COURSE_MASTER c on b.Course_Code = c.Course_Id
where a.Regulation_ID=@pRegulation and a.Institution_ID=@pInsitutionId and a.Branch_Id=@pBranchId and a.YearOfStudy_Id=@pYearofStudyId and a.Semester_Id=@pSemesterId 
 and c.Course_Code not in ('','lab','-','PR','P and','Library')  and a.Flag=0 and b.FLAG=0
FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 
end


if(@pIsOverAllRpt=0)  ---get the section wise report.
begin
SET @query1 = 'select BranchName,[Student Id], UniversityRegNo,StudentName,Section_Name,'+@cols1+',Total
from
(

select * from fnToppersWithSection('+@pInsitutionId+','+@pBranchId+','+@pYearofStudyId+','+@pSemesterId+','+@pExamId+','+@pNoofToppers+','+@pAcYr+')
) as source
pivot(max(marks) for Course_Code in ('+@cols1+')) as pvt order by Section_Name,rank_No'
end
else
begin
SET @query1 = 'select BranchName,[Student Id], UniversityRegNo,StudentName,Section_Name,'+@cols1+',Total
from
(

select * from fnToppersOverAll('+@pInsitutionId+','+@pBranchId+','+@pYearofStudyId+','+@pSemesterId+','+@pExamId+','+@pNoofToppers+','+@pAcYr+')
) as source
pivot(max(marks) for Course_Code in ('+@cols1+')) as pvt order by rank_No'
end


PRINT @query1

execute(@query1)

end



--select * from fnToppersWithSection('+@pInsitutionId+','+@pBranchId+','+@pYearofStudyId+','+@pSemesterId+','+@pExamId+')


--select * from fnToppersWithSection(1,0,2,3,1,3)
--exec Sp_BranchWiseToppers 1,1,2,4,1,3,0,0,4

--select * from T_CFG_COURSE_MASTER where course_code in('cs6301','cs6302','cs6303','cs6304','cs6311','cs6312')





GO
/****** Object:  StoredProcedure [dbo].[SP_FailStudentList]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[SP_FailStudentList]
	@pInstitutionId bigint,
	@pYearId bigint,
	@pBranchId bigint,
	@pExamId bigint
	AS BEGIN
	DECLARE @cols1 AS NVARCHAR(MAX),@query1 AS NVARCHAR(MAX)

	SET @pInstitutionId=1
SET @pYearId=2

SET @cols1 = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_ShortCode)FROM  T_CFG_COURSE_MASTER CM
	join T_MARK_ENTRY ME on CM.Course_Id=ME.Course_Id where ME.YearOfStudy_Id=2 
	and ME.Exam_Id=1 FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,'') 

insert into #table exec SP_NestedFailStudentList @pYearId,@pBranchId,@pExamId output

end
GO
/****** Object:  StoredProcedure [dbo].[sp_getallattendacebyparentId]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE procedure [dbo].[sp_getallattendacebyparentId]
@InstitutionId bigint,
@GroupId bigint,
@parentId bigint,
@studentId bigint,
@Fromdate nvarchar(50),
@todate nvarchar(50)


as
begin

 select * from (select A.StudentTable_Id as StudentID,s.StudentName+' '+s.Initial as StudentName,convert(varchar(50),A.attendance_date,103) as AttendanceDate,A.[Attendance_Type] as AttendanceType,A.[Period],A.[Attendance_Status]  from T_ATTENDANCEENTRY A
inner join T_CFG_STUDENT_PERSONAL_DETAIL s on s.id=A.[StudentTable_Id]

  where convert(varchar(50),attendance_date,103) between @Fromdate and @todate and s.id=@studentId
  ) as pvt
  PIVOT(max([Attendance_Status]) for [Period] IN([1],[2],[3],[4],[5],[6],[7],[8],[9],[10])) as pvt

  end



GO
/****** Object:  StoredProcedure [dbo].[sp_getallAttendancebyperiod]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
create procedure [dbo].[sp_getallAttendancebyperiod]

@COURSE_ID bigint,
@STUDENT_ID bigint

as
begin

select A.StudentName,A.Course_Name,A.No_of_Period,A.Course_Id,A.Id,isnull(AB.Period,0) as 'Absent Period',isnull(P.Period,0) as 'Present Period' from  v_getallcount A 
left outer join v_getallabsentcount AB on A.Id=AB.id
left outer join v_getallpresentcount P on A.Id=P.Id

where A.Course_Id=@COURSE_ID and A.Id=@STUDENT_ID


end
GO
/****** Object:  StoredProcedure [dbo].[sp_getallAttendancereportbydate]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


CREATE procedure [dbo].[sp_getallAttendancereportbydate] -- 0,8,7,13,1,'07/07/2014'

@Staff_Id bigint,
@Branch_Id bigint,
@Year_Id bigint,
@Semester_Id bigint,
@Section_Id bigint,
@Date nvarchar(50)
as
begin
select * from (SELECT stud.SNo,stud.StudentId_Rollno,stud.UniversityRegNo, stud.StudentName+' '+stud.Initial as StudentName, atten.StudentTable_Id,convert(varchar(50),atten.Attendance_Date,103) as Attendance_Date , 
                         atten.Period, atten.Attendance_Status
                      , stud.Branch_Id, stud.YearOfStudy_Id, stud.Semester_Id, stud.Section_Id,stud.Gender
FROM            dbo.T_CFG_STUDENT_PERSONAL_DETAIL stud  JOIN
                         dbo.T_ATTENDANCEENTRY atten (NOLOCK) ON stud.Id = atten.StudentTable_Id
						 where stud.Branch_Id=@Branch_Id and stud.YearOfStudy_Id=@Year_Id 
						and stud.Semester_Id=@Semester_Id and stud.Section_Id=@Section_Id
						 and convert(varchar(50),atten.Attendance_Date,103) =convert(varchar(50),@Date,103) ) as pvt
						 PIVOT (max([Attendance_Status])for [Period] IN([1],[2],[3],[4],[5],[6],[7],[8],[9],[10])) as pvt


						 end




--[sp_getallAttendancereportbydate]  0,8,2,4,1,'13/01/2015'

--select * from T_ATTENDANCEENTRY where Attendance_Date='2014-08-11 00:00:00.000' and YearOfStudy_Id=2 and Section_id=1 and period=2 and Semester_Id=3 and branch_id=8 
GO
/****** Object:  StoredProcedure [dbo].[SP_GetCompanyWiseSelectedStudent]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO



CREATE Procedure [dbo].[SP_GetCompanyWiseSelectedStudent]
 @AcadamicYearID bigint,
 @CompanyID bigint
As
 begin
select (select count(INS.Recruiter_Id) As RecruiterID
 from T_INTERVIEW_STUDENTS INS join  
 T_INTERVIEW_STATUS B on INS.InterviewStatus_Id=B.Id
 join T_RECRUITER R on  Recruiter_Id=R.Id
 join T_INTERVIEW_PROCESS IP on IP.Id =R.Interview_Process_Id
   Where AcademicYear_Id=@AcadamicYearID And Company_Id=@CompanyID
  group by 
  INS.Recruiter_Id) Eligible,
  (   select count(INS.Recruiter_Id) As Appeared
 from T_INTERVIEW_STUDENTS INS join  
 T_INTERVIEW_STATUS B on INS.InterviewStatus_Id=B.Id
 join T_RECRUITER R on  Recruiter_Id=R.Id
 join T_INTERVIEW_PROCESS IP on IP.Id =R.Interview_Process_Id
   Where AcademicYear_Id=@AcadamicYearID And Company_Id=@CompanyID and INS.IsAppeared=1
  group by 
  INS.Recruiter_Id) as Appear,
  (select Description from T_INTERVIEW_STATUS where id=InterviewStatus_Id) as InterviewStatus,
  count(INS.InterviewStatus_Id) InterviewStatusCount
 from T_INTERVIEW_STUDENTS INS join  
 T_INTERVIEW_STATUS S on INS.InterviewStatus_Id=S.Id
 join T_RECRUITER R on  Recruiter_Id=R.Id
 join T_INTERVIEW_PROCESS IP on IP.Id =R.Interview_Process_Id
   Where AcademicYear_Id=@AcadamicYearID And Company_Id=@CompanyID
  group by 
  INS.InterviewStatus_Id,INS.Recruiter_Id
  having  INS.InterviewStatus_Id! =1 
  end



GO
/****** Object:  StoredProcedure [dbo].[sp_getStudentAttendance]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE proc [dbo].[sp_getStudentAttendance](@studid bigint,@attenDate nvarchar(250))
as
begin
select * from (SELECT       stud.SNo,stud.StudentId_Rollno,stud.UniversityRegNo, stud.StudentName+' '+stud.Initial as StudentName, atten.StudentTable_Id,convert(varchar(50),atten.Attendance_Date,103) as Attendance_Date , 
                         atten.Attendance_Type, atten.Period, atten.Attendance_Status,atten.Staff_Id, 
                         atten.Branch_Id, atten.YearOfStudy_Id, atten.Semester_Id, atten.Section_Id
FROM            dbo.T_CFG_STUDENT_PERSONAL_DETAIL stud INNER JOIN
                         dbo.T_ATTENDANCEENTRY atten ON stud.Id = atten.StudentTable_Id
						 where --atten.Branch_Id=@Branch_Id and atten.YearOfStudy_Id=@Year_Id 

						--and atten.Semester_Id=@Semester_Id and atten.Section_Id=@Section_Id
						stud.Id=@studid
						 and convert(varchar(50),atten.Attendance_Date,103) =@attenDate ) as pvt
						 PIVOT (max([Attendance_Status])for [Period] IN([1],[2],[3],[4],[5],[6],[7],[8],[9],[10])) as pvt

						 end
GO
/****** Object:  StoredProcedure [dbo].[SP_NestedFailStudentList]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
	
	CREATE PROCEDURE [dbo].[SP_NestedFailStudentList]

	@ptYearId bigint,@ptBranchId bigint,@ptSemId bigint,@pExamId bigint,@AcadamicYearId bigint

	As
	BEGIN

	Declare @DBQuery nvarchar(MAX),@appendInMarkInfo nvarchar(MAX)
	,@appendInStuCountInfo nvarchar(MAX),@cols1 nvarchar(MAX),@Query nvarchar(MAX),@vAcademicYrId bigint,@passmark nvarchar(5);

	set @passmark=(select SettingValue from T_CFG_CONFIGURATON_SETTINGS where SettingName='PassMark');
	SET @vAcademicYrId=(select ID from T_CFG_ACADEMIC_YEAR where Academic_year_Status=1 and Status=1 and flag=0);

	SET @cols1 = STUFF((SELECT distinct ',' + QUOTENAME(CM.Course_ShortCode)FROM  T_CFG_COURSE_MASTER CM
	join T_MARK_ENTRY ME (NOLOCK) on CM.Course_Id=ME.Course_Id where ME.YearOfStudy_Id=@ptYearId and ME.Semester_Id=@ptSemId
	and ME.Exam_Id=@pExamId and ME.Branch_Id=@ptBranchId and ME.AcademicYear_Id=@AcadamicYearId FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,''); 
	--set @pYearId=1;SET @pBranchId=6;
	--if(@ptYearId=1)
	--	Begin
	--		SET @appendInMarkInfo='YearOfStudy_Id='+cast(@ptYearId as nvarchar(5))+' and Exam_Id='+cast(@pExamId as nvarchar(5));
	--		SET @appendInStuCountInfo='stu.YearOfStudy_Id='+cast(@ptYearId  as nvarchar(5));
		
	--	END
	--ELSE
	--	BEGIN
			SET @appendInMarkInfo= 'YearOfStudy_Id='+cast(@ptYearId as nvarchar(5))+' and Exam_Id='+cast(@pExamId as nvarchar(5))+' and ME.Branch_Id='+cast(@ptBranchId as nvarchar(5)) +' and ME.Semester_Id='+cast(@ptSemId as nvarchar(5))+' and ME.AcademicYear_Id='+cast(@AcadamicYearId as nvarchar(5));
			SET @appendInStuCountInfo='stu.YearOfStudy_Id='+cast(@ptYearId  as nvarchar(5))+' and STU.Branch_Id='+cast(@ptBranchId as nvarchar(5))+' and STU.Semester_Id='+cast(@ptSemId as nvarchar(5))+' and STU.AcademicYear_Id='+cast(@vAcademicYrId as nvarchar(5)) ;
		--END

	SET @DBQuery=';with markInfo(Branch_Id,Section_Id,Course_ID,Branch_Code,section_name,Course_ShortCode,Absentees,Failures,OD)
	as
	(
	select  ME.Branch_Id,ME.Section_Id,ME.Course_ID,B.Branch_Code,section_name,CM.Course_ShortCode,sum(case when ME.Is_Absent=1 then 1 else 0 end) Absentees,
	sum(case when ME.Marks<'+@passmark+' and ME.Is_Absent=0 and (ME.IS_OD=0 or ME.Is_OD is null) then 1 else 0 end) Failures,sum(case when ME.Is_OD=1 then 1 else 0 end) OD
	--cast(Marks as int) as Marks, CM.Course_ShortCode 
	from T_MARK_ENTRY ME (NOLOCK) 
	join T_CFG_COURSE_MASTER CM on ME.Course_Id = CM.Course_Id
	join T_CFG_BRANCH B on ME.Branch_Id = B.Branch_Id
	join T_CFG_SECTION_MASTER S on ME.Section_Id = S.ID
	where '+ @appendInMarkInfo +'  group by B.Branch_Code,section_name,
	Course_ShortCode,ME.Branch_Id,ME.Section_Id,ME.Course_ID
	--order by B.Branch_Code,section_name,CM.Course_ShortCode

	)
	,
	stuCountInfo(Branch_Code,Section_Name,STRENGTH)
	as
	(

	select br.Branch_Code,sec.Section_Name ,count(stu.id) as STRENGTH from T_CFG_STUDENT_PERSONAL_DETAIL stu
	join T_CFG_BRANCH br on stu.Branch_Id=br.Branch_Id
	join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
	where '+@appendInStuCountInfo +' and stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0
	group by br.Branch_Code,sec.Section_Name
	--order by br.Branch_Code,sec.Section_Name
	)

	'

	--select @appendQuery
	--print(@DBQuery)

	--exec(@DBQuery)

	set @Query=''+@DBQuery+' select [BRANCH ID], [SECTION ID], BRANCH,SECTION,STRENGTH,'+@cols1+'
	from
	(

	select a.Branch_Id as [BRANCH ID],a.Section_Id as [SECTION ID], a.Branch_code as BRANCH,a.section_name as SECTION,b.STRENGTH,a.Course_ShortCode,
	--(case when (a.absentees > 0 ) then cast(CONCAT(cast(a.Failures as nvarchar(10)),''+'',cast(a.Absentees as nvarchar(10)), ''(AB)'')as nvarchar(10)) else cast((a.Failures)as nvarchar(10)) end) as MARKS
	(case 
		when (a.Failures>0 and a.Absentees=0 and a.OD=0) then cast((cast(a.Failures as nvarchar(10))) +''|''+cast(a.Course_Id as nvarchar(10)) as nvarchar(100))   /*only failures F*/
		when (a.Failures>0 and a.Absentees>0 and a.OD=0) then cast(CONCAT(cast(a.Failures as nvarchar(10)),''+'',cast(a.Absentees as nvarchar(10)), ''(AB)'')+''|''+cast(a.Course_Id as nvarchar(10)) as nvarchar(100)) /*F/Ab */
		when (a.Failures>0 and a.OD>0 and a.Absentees=0) then cast(CONCAT(cast(a.Failures as nvarchar(10)),''+'',cast(a.OD as nvarchar(10)), ''(OD)'') +''|''+cast(a.Course_Id as nvarchar(10)) as nvarchar(100))		 /*F/od */
		when (a.Absentees>0 and a.Failures=0 and a.OD=0) then cast(CONCAT(cast(a.Absentees as nvarchar(10)), ''(AB)'') +''|''+cast(a.Course_Id as nvarchar(10)) as nvarchar(100))  /* only absentees AB	*/		
		when (a.Absentees>0 and a.OD>0 and a.Failures=0) then cast(CONCAT(cast(a.Absentees as nvarchar(10)), ''(AB)'',''+'',cast(a.OD as nvarchar(10)),''(OD)'') +''|''+cast(a.Course_Id as nvarchar(10)) as nvarchar(100))	/* AB/OD */	
		when (a.OD>0 and a.Failures=0 and a.Absentees=0) then cast(CONCAT(cast(a.OD as nvarchar(10)), ''(OD)'') +''|''+cast(a.Course_Id as nvarchar(10)) as nvarchar(100))  /* only OD */	
		when (a.Failures>0 and a.Absentees>0 and a.OD>0) then cast(CONCAT(cast(a.Failures as nvarchar(10)),''+'',cast(a.Absentees as nvarchar(10)), ''(AB)'',''+'',cast(a.OD as nvarchar(10)), ''(OD)'') +''|''+cast(a.Course_Id as nvarchar(10)) as nvarchar(100)) /*all the three  fail/Ab/OD */
		when (a.Failures=0 and a.Absentees=0 and a.OD=0) then ''0''   /*0*/
	 END
	 )as MARKS

	
	from markInfo a
	join stuCountInfo b on a.Branch_Code=b.Branch_Code
	where a.section_name=b.Section_Name
	) as Source
	pivot(max(MARKS) for Course_ShortCode in ('+@cols1+')) as pvt
	 order by BRANCH'

	--print(@Query)
	exec(@Query)

	END

	--exec SP_NestedFailStudentList 2,1,4,1
	--select markInfo.*,stuCountInfo.STRENGTH from markInfo
	--join stuCountInfo on markInfo.Branch_Code=stuCountInfo.Branch_Code
	--where markInfo.section_name=stuCountInfo.Section_Name



	--select * from T_MARK_ENTRY where Branch_Id=1 and YearOfStudy_Id=2 and Section_Id=2 and 

	--select * from T_CFG_COURSE_MASTER where Course_ShortCode='DBMS'


	

	
	

	
	

	
	
	
	

	
GO
/****** Object:  StoredProcedure [dbo].[SP_NestedFailuresSubWise]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[SP_NestedFailuresSubWise]

 @pInstId nvarchar(5),@pYearId nvarchar(5),@pSemId nvarchar(5),@pExamId nvarchar(5),@AcademicYrId nvarchar(5)

AS BEGIN

DECLARE 
@cols1 NVARCHAR(MAX),
@query1 NVARCHAR(MAX),
--@AcademicYrId nvarchar(5),
@passmark nvarchar(5)

set @passmark=(select SettingValue from T_CFG_CONFIGURATON_SETTINGS where SettingName='PassMark');
--SET @AcademicYrId=(select ID from T_CFG_ACADEMIC_YEAR where flag=0 and Academic_year_Status=1)

	SET @cols1 = STUFF((SELECT  distinct ',' + QUOTENAME(cast(count(ME.StudentTable_Id) as nvarchar(3)) + ' SUBJECT') 
	from T_MARK_ENTRY ME (NOLOCK)
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	join T_CFG_BRANCH BM on ME.Branch_Id=BM.Branch_Id
	join T_CFG_SECTION_MASTER SM on ME.Section_Id = SM.ID
	where ME.Institution_Id=@pInstId and ME.AcademicYear_Id=@AcademicYrId and ME.YearOfStudy_Id=@pYearId and ME.Semester_Id=@pSemId and ME.Marks<@passmark and Me.Exam_Id=@pExamId
	group by ME.StudentTable_Id 
	FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '')


SET @query1 = 'select Branch_Id as BranchId,ID as SectionId,Branch_Name +'' - '' +	Section_Name BRANCH,'+@cols1+'
	from
	(select Branch_Id,Branch_Name,ID,Section_Name,NoOfSub,count(StudentTable_Id) failCount
		from(select BM.Branch_Id,BM.Branch_Name,SM.ID,SM.Section_Name, cast(count(ME.StudentTable_Id) as nvarchar(3)) + '' SUBJECT'' NoOfSub,ME.StudentTable_Id from T_MARK_ENTRY ME (NOLOCK)
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	join T_CFG_BRANCH BM on ME.Branch_Id=BM.Branch_Id
	join T_CFG_SECTION_MASTER SM on ME.Section_Id = SM.ID
	where ME.Institution_Id='+@pInstId+' and ME.AcademicYear_Id='+@AcademicYrId+' and  ME.YearOfStudy_Id='+@pYearId+' and ME.Semester_Id='+@pSemId+' and ME.Marks<'+@passmark+' and Me.Exam_Id='+@pExamId+'
	group by ME.StudentTable_Id ,BM.Branch_Id,BM.Branch_Name,SM.ID,SM.Section_Name) as t
	group by NoOfSub, Branch_Id,Branch_Name,ID,Section_Name) as source
	pivot (max(failCount) for NoOfSub in ('+@cols1+')) as pvt order by BRANCH'

	print (@cols1)
	print(@query1)
	EXEC(@query1) 

END

--exec SP_NestedFailuresSubWise 1,2,3,1



GO
/****** Object:  StoredProcedure [dbo].[SP_NestedFailuresSubWiseDtl]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE PROCEDURE [dbo].[SP_NestedFailuresSubWiseDtl]
@pInstId nvarchar(5),
@pBranchId nvarchar(5),
@pYearId nvarchar(5),
@pSemId nvarchar(5),
@pSectionId nvarchar(5),
@pExamId nvarchar(5),
@pNoOfSub nvarchar(5),
@AcademicYrId nvarchar(5)

AS BEGIN

DECLARe @passmark int
--SET @AcademicYrId=(select ID from T_CFG_ACADEMIC_YEAR where flag=0 and Academic_year_Status=1);
set @passmark=(select SettingValue from T_CFG_CONFIGURATON_SETTINGS where SettingName='PassMark');
WITH STUFF1 AS 
(
	SELECT BR.Branch_Code, Y.YearOfStudy_Name, SEM.Semester_Name, SEC.Section_Name, CM.Course_ShortCode, ME.StudentTable_Id, PD.StudentName+' '+PD.Initial as StudentName 
	FROM T_MARK_ENTRY ME (NOLOCK)
	JOIN T_CFG_BRANCH BR ON ME.Branch_Id = BR.Branch_Id
	JOIN T_CFG_YEAROFSTUDY_MASTER Y ON ME.YearOfStudy_Id = Y.YearOfStudy_Id
	JOIN T_CFG_SEMESTER SEM ON ME.Semester_Id = SEM.Semester_Id
	JOIN T_CFG_SECTION_MASTER SEC ON ME.Section_Id = SEC.ID
	JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
	JOIN T_CFG_STUDENT_PERSONAL_DETAIL PD ON ME.StudentTable_Id = PD.Id
	WHERE ME.Institution_Id = @pInstId and ME.AcademicYear_Id = @AcademicYrId and ME.Branch_Id = @pBranchId and  ME.YearOfStudy_Id = @pYearId 
	and ME.Semester_Id = @pSemId and ME.Section_Id = @pSectionId  and ME.Exam_Id = @pExamId and ME.Marks < @passmark
	GROUP BY ME.StudentTable_Id, CM.Course_ShortCode, PD.StudentName,PD.Initial, BR.Branch_Code, Y.YearOfStudy_Name, SEM.Semester_Name, SEC.Section_Name
)
    SELECT StudentName, CourseNameList, StudentName, Branch_Code, YearOfStudy_Name, Semester_Name, Section_Name  FROM (SELECT StudentTable_Id, StudentName, Branch_Code, YearOfStudy_Name, Semester_Name, Section_Name,
	STUFF((SELECT ', ' + CAST(Course_ShortCode AS VARCHAR(10)) [text()] FROM (SELECT * FROM STUFF1)T
	WHERE StudentTable_Id = T1.StudentTable_Id
	FOR XML PATH(''), TYPE).value('.','NVARCHAR(MAX)'),1,2,'') CourseNameList
	FROM (SELECT * FROM STUFF1)T1
	GROUP BY StudentTable_Id, StudentName, StudentName, Branch_Code, YearOfStudy_Name, Semester_Name, Section_Name)src WHERE (SELECT count(s) FROM SplitString(CourseNameList,',')SL) = @pNoOfSub
END

--EXEC SP_NestedFailuresSubWiseDtl 1,3,2,3,1,1,3

GO
/****** Object:  StoredProcedure [dbo].[SP_PassFailPercentage]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[SP_PassFailPercentage]

 @pInstId nvarchar(5),@pYearId nvarchar(5),@pSemId nvarchar(5),@pExamId nvarchar(5),@AcademicYrId nvarchar(5)

AS BEGIN

DECLARE 
@cols1 NVARCHAR(MAX),
@query1 NVARCHAR(MAX),

@passmark int


set @passmark=(select SettingValue from T_CFG_CONFIGURATON_SETTINGS where SettingName='PassMark')

 
--FAIL
;With NA(NA,BranchId,SectionId,BRANCH)
as
(
 --Appeare

select DISTINCT  count(M.StudentTable_id )NA,
B.Branch_Id as BranchId,s.ID as SectionId, B.Branch_Code+ ' - ' +S.Section_Name AS BRANCH
from T_MARK_ENTRY M (NOLOCK)
join T_CFG_BRANCH B ON M.Branch_Id =B.Branch_Id
JOIN T_CFG_SECTION_MASTER S ON M.Section_Id=S.ID WHERE M.AcademicYear_Id=@AcademicYrId and
M.Exam_Id=@pExamId AND M.YearOfStudy_Id=@pYearId AND M.Semester_Id=@pSemId AND M.Is_Absent=0 

GROUP BY B.Branch_Code,S.Section_Name ,b.Branch_id,s.ID 
),
 NF(NF,BranchId,SectionId,BRANCH)
AS
(
select DISTINCT  count(M.StudentTable_id )NF,
B.Branch_Id as BranchId,s.ID as SectionId, B.Branch_Code+ ' - ' +S.Section_Name AS BRANCH
from T_MARK_ENTRY M (NOLOCK)
join T_CFG_BRANCH B ON M.Branch_Id =B.Branch_Id
JOIN T_CFG_SECTION_MASTER S ON M.Section_Id=S.ID WHERE M.AcademicYear_Id=@AcademicYrId and
M.Exam_Id=@pExamId AND M.YearOfStudy_Id=@pYearId AND M.Semester_Id=@pSemId AND M.Is_Absent=0  and M.Marks < @passmark

GROUP BY B.Branch_Code,S.Section_Name ,b.Branch_id,s.ID 
)

,NP(NP,BranchId,SectionId,BRANCH)
as
(
--PASS

select DISTINCT  count(M.StudentTable_id )NP,
B.Branch_Id as BranchId,s.ID as SectionId, B.Branch_Code+ ' - ' +S.Section_Name AS BRANCH
from T_MARK_ENTRY M (NOLOCK)
join T_CFG_BRANCH B ON M.Branch_Id =B.Branch_Id
JOIN T_CFG_SECTION_MASTER S ON M.Section_Id=S.ID WHERE M.AcademicYear_Id=@AcademicYrId and
M.Exam_Id=@pExamId AND M.YearOfStudy_Id=@pYearId AND M.Semester_Id=@pSemId AND M.Is_Absent=0  and M.Marks>= @passmark

GROUP BY B.Branch_Code,S.Section_Name ,b.Branch_id,s.ID 
)

select NA.BranchId,NA.SectionId,NA.BRANCH,Na.NA,NP.NP,Nf.NF,convert(decimal(18,2),(CAST(NP.NP as decimal)*100)/CAST(NA.NA as decimal)) as [Pass %] from NA
join NP on NA.BranchId=NP.BranchId
join NF on NF.BranchId=NA.BranchId
where NA.SectionId=NP.SectionId and na.SectionId=nf.SectionId

--select DISTINCT  B.Branch_Id as BranchId,s.ID as SectionId, B.Branch_Code+ ' - ' +S.Section_Name AS BRANCH,
--(SELECT COUNT(ID) FROM T_CFG_STUDENT_PERSONAL_DETAIL WHERE TC_ISSUED=0 AND Status=1 AND Delete_Flag=0 AND  YearOfStudy_Id=@pYearId AND Semester_Id=@pSemId AND Branch_Id=b.Branch_Id and Section_Id=s.ID and AcademicYear_Id=@AcademicYrId) AS NA,
--(select count(StudentTable_Id) failCount
--		from(select ME.StudentTable_Id from T_MARK_ENTRY ME
--	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
--	join T_CFG_BRANCH BM on ME.Branch_Id=BM.Branch_Id
	
--	where ME.Institution_Id=@pInstId and ME.AcademicYear_Id=@AcademicYrId and  ME.YearOfStudy_Id=@pYearId and ME.Semester_Id=@pSemId and
--	 ME.Marks > 50 and Me.Exam_Id=@pExamId and Section_Id=s.ID
--	and ME.Branch_id= B.Branch_id
--	group by ME.StudentTable_Id ,BM.Branch_Name) as t
--	) as NP,
	
--	(convert(int,(SELECT COUNT(ID) FROM T_CFG_STUDENT_PERSONAL_DETAIL WHERE TC_ISSUED=0 AND Status=1 AND Delete_Flag=0 AND  
--	YearOfStudy_Id=@pYearId AND Semester_Id=@pSemId AND Branch_Id=b.Branch_Id and Section_Id=s.ID))-convert(int,(
--     select count(StudentTable_Id) failCount from
--	 (select ME.StudentTable_Id from T_MARK_ENTRY ME
--	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
--	join T_CFG_BRANCH BM on ME.Branch_Id=BM.Branch_Id
	
--	where ME.Institution_Id=@pInstId and ME.AcademicYear_Id=@AcademicYrId and  ME.YearOfStudy_Id=@pYearId and ME.Semester_Id=@pSemId and ME.Marks<50 and
--	 Me.Exam_Id=@pExamId and Section_Id=s.ID
--	and ME.Branch_id= B.Branch_id
--	group by ME.StudentTable_Id ,BM.Branch_Name) as t
--	))) as NF,
-- convert(decimal(18,2),(convert(decimal(18,2),(select count(StudentTable_Id) failCount
--		from(select ME.StudentTable_Id from T_MARK_ENTRY ME
--	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
--	join T_CFG_BRANCH BM on ME.Branch_Id=BM.Branch_Id
	
--	where ME.Institution_Id=@pInstId and ME.AcademicYear_Id=@AcademicYrId and  ME.YearOfStudy_Id=@pYearId and ME.Semester_Id=@pSemId and ME.Marks<50 and Me.Exam_Id=@pExamId and Section_Id=s.ID
--	and ME.Branch_id= B.Branch_id
--	group by ME.StudentTable_Id ,BM.Branch_Name) as t))*100/(SELECT COUNT(ID) FROM T_CFG_STUDENT_PERSONAL_DETAIL WHERE TC_ISSUED=0 AND Status=1 AND Delete_Flag=0 AND  YearOfStudy_Id=@pYearId AND Semester_Id=@pSemId AND Branch_Id=b.Branch_Id and Section_Id=s.ID))) As [Pass %]
	

-- from T_MARK_ENTRY M 
--join T_CFG_BRANCH B ON M.Branch_Id =B.Branch_Id
--JOIN T_CFG_SECTION_MASTER S ON M.Section_Id=S.ID WHERE M.Exam_Id=@pExamId AND M.YearOfStudy_Id=@pYearId AND M.Semester_Id=@pSemId AND M.Is_Absent=0
----AND (B.Branch_Code+ ' - ' +S.Section_Name!='CIVIL - A') cmt by NAGRAJ
--GROUP BY B.Branch_Code,S.Section_Name ,b.Branch_id,s.ID

END

--exec [SP_PassFailPercentage] 1,3,6,1,4





GO
/****** Object:  StoredProcedure [dbo].[sp_PlacementReport]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[sp_PlacementReport]
@AcademicYearId nvarchar(50),
@CompanyIds nvarchar(50),
@InterviewStatusId nvarchar(50)
as
begin
declare @cols nvarchar(max);
declare @cols1 nvarchar(max);
declare @query nvarchar(max);

set @cols = stuff((select distinct ',' + QUOTENAME(B.Branch_Code) 
from T_RECRUITER R
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,',') 
where R.AcademicYear_Id=@AcademicYearId 
and R.Company_Id in (select s from SplitString(@CompanyIds,','))) 
FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '');

set @cols1 = stuff((select distinct '+ISNULL(' + QUOTENAME(B.Branch_Code) + ',0)' 
from T_RECRUITER R
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,',') 
where R.AcademicYear_Id=@AcademicYearId
and R.Company_Id in (select s from SplitString(@CompanyIds,','))) 
FOR XML PATH(''), TYPE).value('.', 'NVARCHAR(MAX)'), 1, 1, '') ; 

if (@InterviewStatusId!=1)
begin
set @query = 'select Company_Name ''Company Name'', [From Date], [To Date], '+@cols+', ('+@cols1+') as ''Total'' 
from
(select C.Company_Name, CONVERT(NVARCHAR(20),R.From_Date,103) as [From Date], CONVERT(NVARCHAR(20),R.To_Date,103) as [To Date], 
B.Branch_Code, S.StudentName 
from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,'','')) 
join T_CFG_STUDENT_PERSONAL_DETAIL S on B.Branch_Id = S.Branch_Id
join T_INTERVIEW_STUDENTS ISTU on R.Id = ISTU.Recruiter_Id
where S.Id = ISTU.Student_Id 
and ISTU.InterviewStatus_Id = '+@InterviewStatusId+'
and R.AcademicYear_Id='+@AcademicYearId+' 
and R.Company_Id in (select s from SplitString('''+@CompanyIds+''','',''))
and ISTU.IsAppeared=1
)src pivot (count(StudentName) for Branch_Code in('+@cols+')) as PVT
union
select ''Total'','''','''', '+@cols+', ('+@cols1+') as ''Total'' 
from
(select B.Branch_Code, count(B.Branch_Code) as ''Total'' 
from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,'','')) 
join T_CFG_STUDENT_PERSONAL_DETAIL S on B.Branch_Id = S.Branch_Id
join T_INTERVIEW_STUDENTS ISTU on R.Id = ISTU.Recruiter_Id
where S.Id = ISTU.Student_Id and ISTU.InterviewStatus_Id ='+ @InterviewStatusId+' 
and R.AcademicYear_Id='+@AcademicYearId+' 
and R.Company_Id in (select s from SplitString('''+@CompanyIds+''','','')) 
and ISTU.IsAppeared=1 
group by B.Branch_Code
)src pivot (Max(Total) for Branch_Code in('+@cols+')) as PVT'
execute(@query);

select C.Company_Name 'Company Name', sum(case ISTU.IsAppeared when '1' then 1 else 0 end) 'Appeared', count(ISTU.Recruiter_Id) 'Eligible' 
from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,',')) 
join T_CFG_STUDENT_PERSONAL_DETAIL S on B.Branch_Id = S.Branch_Id
join T_INTERVIEW_STUDENTS ISTU on R.Id = ISTU.Recruiter_Id
where S.Id = ISTU.Student_Id 
and R.AcademicYear_Id=@AcademicYearId 
and R.Company_Id in (select s from SplitString(@CompanyIds,',')) 
group by C.Company_Name

;with cte as(
select C.Company_Name, B.Branch_Code from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,',')))

Select Main.Company_Name, Left(Main.Students,Len(Main.Students)-1)+',Total' As "Students"
From
(Select distinct ST2.Company_Name, 
 (Select ST1.Branch_Code + ',' AS [text()]
                From cte ST1
                Where ST1.Company_Name = ST2.Company_Name
                ORDER BY ST1.Company_Name
                For XML PATH ('')
            ) [Students]
        From cte ST2
 )[Main]


select C.Company_Name 'Company Name',S.StudentId_Rollno as 'Student Id' , 
S.UniversityRegNo  'Register No'
, S.StudentName Name,B.Branch_Code+'/'+Y.YearOfStudy_Name+'/'+SEC.Section_Name as 'Class',B.Branch_Code 'Branch',
ISTA.Description 'Status'

 
from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,',')) 
join T_CFG_STUDENT_PERSONAL_DETAIL S on B.Branch_Id = S.Branch_Id
join T_INTERVIEW_STUDENTS ISTU on R.Id = ISTU.Recruiter_Id
join T_INTERVIEW_STATUS ISTA on ISTU.InterviewStatus_Id = ISTA.Id
join T_CFG_YEAROFSTUDY_MASTER Y on S.YearOfStudy_Id = Y.YearOfStudy_Id
join T_CFG_SECTION_MASTER SEC on S.Section_Id = SEC.ID
where S.Id = ISTU.Student_Id 
and ISTU.InterviewStatus_Id = @InterviewStatusId 
and R.AcademicYear_Id=@AcademicYearId 
and R.Company_Id in (select s from SplitString(@CompanyIds,','))
and ISTU.IsAppeared=1
end
else
begin
set @query = 'select Company_Name ''Company Name'', [From Date], [To Date], '+@cols+', ('+@cols1+') as ''Total'' 
from
(select C.Company_Name, CONVERT(NVARCHAR(20),R.From_Date,103) as [From Date], CONVERT(NVARCHAR(20),R.To_Date,103) as [To Date], 
B.Branch_Code, S.StudentName 
from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,'','')) 
join T_CFG_STUDENT_PERSONAL_DETAIL S on B.Branch_Id = S.Branch_Id
join T_INTERVIEW_STUDENTS ISTU on R.Id = ISTU.Recruiter_Id
where S.Id = ISTU.Student_Id
and R.AcademicYear_Id='+@AcademicYearId+' 
and R.Company_Id in (select s from SplitString('''+@CompanyIds+''','',''))
)src pivot (count(StudentName) for Branch_Code in('+@cols+')) as PVT
union
select ''Total'','''','''', '+@cols+', ('+@cols1+') as ''Total'' from
(select B.Branch_Code, count(B.Branch_Code) as ''Total'' 
from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,'','')) 
join T_CFG_STUDENT_PERSONAL_DETAIL S on B.Branch_Id = S.Branch_Id
join T_INTERVIEW_STUDENTS ISTU on R.Id = ISTU.Recruiter_Id
where S.Id = ISTU.Student_Id 
and R.AcademicYear_Id='+@AcademicYearId+' 
and R.Company_Id in (select s from SplitString('''+@CompanyIds+''','','')) 
group by B.Branch_Code
)src pivot (Max(Total) for Branch_Code in('+@cols+')) as PVT'
execute(@query);

select C.Company_Name 'Company Name', sum(case ISTU.IsAppeared when '1' then 1 else 0 end) 'Appeared', count(ISTU.Recruiter_Id) 'Eligible' 
from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,',')) 
join T_CFG_STUDENT_PERSONAL_DETAIL S on B.Branch_Id = S.Branch_Id
join T_INTERVIEW_STUDENTS ISTU on R.Id = ISTU.Recruiter_Id
where S.Id = ISTU.Student_Id and R.AcademicYear_Id=@AcademicYearId 
and R.Company_Id in (select s from SplitString(@CompanyIds,',')) 
group by C.Company_Name

;with cte as(
select C.Company_Name, B.Branch_Code from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,',')))

Select Main.Company_Name, Left(Main.Students,Len(Main.Students)-1)+',Total' As "Students"
From
(Select distinct ST2.Company_Name, 
            (Select ST1.Branch_Code + ',' AS [text()]
                From cte ST1
                Where ST1.Company_Name = ST2.Company_Name
                ORDER BY ST1.Company_Name
                For XML PATH ('')
            ) [Students]
        From cte ST2
)[Main]

select C.Company_Name 'Company Name', 
 S.StudentId_Rollno as 'Student Id' ,S.UniversityRegNo 'Register No' , S.StudentName 'Name'
 ,B.Branch_Code+'/'+Y.YearOfStudy_Name+'/'+SEC.Section_Name as 'Class'
 ,B.Branch_Code 'Branch', ISTA.Description 'Status'
from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,',')) 
join T_CFG_STUDENT_PERSONAL_DETAIL S on B.Branch_Id = S.Branch_Id
join T_INTERVIEW_STUDENTS ISTU on R.Id = ISTU.Recruiter_Id
join T_INTERVIEW_STATUS ISTA on ISTU.InterviewStatus_Id = ISTA.Id
join T_CFG_YEAROFSTUDY_MASTER Y on S.YearOfStudy_Id = Y.YearOfStudy_Id
join T_CFG_SECTION_MASTER SEC on S.Section_Id = SEC.ID
where S.Id = ISTU.Student_Id 
and R.AcademicYear_Id=@AcademicYearId 
and R.Company_Id in (select s from SplitString(@CompanyIds,','))
end
end


--exec sp_PlacementReport 4,'1,2',5
GO
/****** Object:  StoredProcedure [dbo].[sp_SaveOdAttendance]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[sp_SaveOdAttendance]
@plstStudentTableId NVARCHAR(MAX),
@plstPeriods NVARCHAR(MAX),
@plstAttendanceDate NVARCHAR(MAX),
@pFromDate NVARCHAR(150),
@pToDate NVARCHAR(150),
@pIsAttendance BIT,
@pRemarks NVARCHAR(500),
@plstSectionIds NVARCHAR(MAX),
@pOdHdrId BIGINT,
@pUserId BIGINT,
@pUserName NVARCHAR(250),
@pOdHdrIdOut BIGINT OUTPUT
AS
BEGIN

DECLARE
@BranchId BIGINT,
@YearId BIGINT,
@SemesterId BIGINT,
@SectionId BIGINT,
@strSectionId NVARCHAR(10),
@BatchId BIGINT,
@StudentTableId BIGINT,
@Period INT,
@AttendanceExists INT,
@AttendanceDate DATETIME,
@Od_Hdr_Id BIGINT,
@Staff_id bigint=0,
@Course_id bigint=0,
@StaffExists bigint=0,
@ElectiveBatch bigint=0,
@AcademicYearId bigint

set @AcademicYearId=(select ID from T_CFG_ACADEMIC_YEAR where flag=0 and Academic_year_Status=1)

IF(@pOdHdrId = 0)
BEGIN

INSERT INTO T_OD_DETAILS_HDR (Common_Od, From_Date, To_Date, OD_Periods, OD_Remarks, Delete_Flag, Status, Created_Date, Created_By)
VALUES (@pIsAttendance, CONVERT(DATETIME, @pFromDate, 120), CONVERT(DATETIME, @pToDate, 120), @plstPeriods, @pRemarks, 0, 1, getdate(), @pUserName)

SET @Od_Hdr_Id = SCOPE_IDENTITY();
SET @pOdHdrIdOut = @Od_Hdr_Id;

END
ELSE
BEGIN

SET @Od_Hdr_Id = @pOdHdrId;
SET @pOdHdrIdOut = @Od_Hdr_Id;

END--Save Header

DECLARE @CursorStudentId CURSOR
SET @CursorStudentId = CURSOR FAST_FORWARD
FOR
SELECT s FROM (Select s from SplitString(@plstStudentTableId,','))T1

OPEN @CursorStudentId

FETCH NEXT FROM @CursorStudentId
INTO @StudentTableId

WHILE @@FETCH_STATUS = 0
BEGIN

INSERT INTO T_OD_DETAILS_DTL (ODHdr_Id, StudentTable_Id, OD_Approve, Delete_Flag)
VALUES(@Od_Hdr_Id, @StudentTableId, @pIsAttendance, 0)

FETCH NEXT FROM @CursorStudentId
INTO @StudentTableId

END

CLOSE @CursorStudentId  
DEALLOCATE @CursorStudentId

-----------------------SAVE ATTENDANCE START
IF(@pIsAttendance = 1)
BEGIN

-----------------------DATE LOOP START
DECLARE @CursorDate CURSOR
SET @CursorDate = CURSOR FAST_FORWARD
FOR
SELECT s FROM (Select s from SplitString(@plstAttendanceDate,','))T0

OPEN @CursorDate

FETCH NEXT FROM @CursorDate
INTO @AttendanceDate

WHILE @@FETCH_STATUS = 0
BEGIN

SET @CursorStudentId = CURSOR FAST_FORWARD
FOR
SELECT s FROM (Select s from SplitString(@plstStudentTableId,','))T1

OPEN @CursorStudentId

FETCH NEXT FROM @CursorStudentId
INTO @StudentTableId

WHILE @@FETCH_STATUS = 0
BEGIN

SET @BranchId = (Select branch_id from T_CFG_STUDENT_PERSONAL_DETAIL where id = @StudentTableId)
SET @YearId = (Select YearOfStudy_Id from T_CFG_STUDENT_PERSONAL_DETAIL where id = @StudentTableId)
SET @SemesterId = (Select Semester_Id from T_CFG_STUDENT_PERSONAL_DETAIL where id = @StudentTableId)
SET @SectionId = (Select Section_Id from T_CFG_STUDENT_PERSONAL_DETAIL where id = @StudentTableId)
SET @BatchId = (Select LabBatch_Id from T_CFG_STUDENT_PERSONAL_DETAIL where id = @StudentTableId)
SET @ElectiveBatch=(Select Elective_Batch_Id from T_CFG_STUDENT_PERSONAL_DETAIL where id = @StudentTableId)
----------------------PERIOD LOOP START

DECLARE @CursorPeriod CURSOR
SET @CursorPeriod = CURSOR FAST_FORWARD
FOR
SELECT s FROM (Select s from SplitString(@plstPeriods,','))T2

OPEN @CursorPeriod

FETCH NEXT FROM @CursorPeriod
INTO @Period

WHILE @@FETCH_STATUS = 0
BEGIN

SET @AttendanceExists = (Select COUNT(*) from T_ATTENDANCEENTRY where Attendance_Date = CONVERT(DATETIME, @AttendanceDate,120) and StudentTable_Id = @StudentTableId
												and Period = @Period)

IF(@AttendanceExists = 0)
BEGIN

SET  @Course_id=(select course_Id from T_CFG_TIMETABLE_DEFINITION where AcademicYear_Id=@AcademicYearId and Branch_Id=@BranchId and YearOfStudy_Id=@YearId and Semester_Id=@SemesterId and Section_Id=@SectionId and Day=DATENAME(dw,CONVERT(DATETIME, @AttendanceDate,120)) and Delete_Flag=0 and Period=@Period)


set @StaffExists=( select COUNT(*)   from T_STAFF_ASSIGINING_CLASS_COURSE_HDR h 
join T_STAFF_ASSIGINING_CLASS_COURSE_DTL d on h.Id=d.Hdr_Id
where AcademicYear_Id=@AcademicYearId and Branch_Id=@BranchId and YearOfStudy_Id=@YearId and Semester_Id=@SemesterId and Class_Section_Id=@SectionId 
and Course_Id in(@Course_id ))

IF(@StaffExists != 0)
begin

set @Staff_id=(select top 1 Staff_id  from T_STAFF_ASSIGINING_CLASS_COURSE_HDR h 
join T_STAFF_ASSIGINING_CLASS_COURSE_DTL d on h.Id=d.Hdr_Id
where h.AcademicYear_Id=@AcademicYearId and h.Branch_Id=@BranchId and h.YearOfStudy_Id=@YearId and h.Semester_Id=@SemesterId and d.Class_Section_Id=@SectionId
and d.Course_Id in( @Course_id))

end


INSERT INTO T_ATTENDANCEENTRY (Academic_Year,Branch_Id, YearOfStudy_Id, Semester_Id, Section_Id, Batch_Id,Elective_Batch_Id, Attendance_Date, Attendance_Day, Attendance_Status, 
Staff_Id, StudentTable_Id, Course_Id, Period, TopicsCovered, [Status], Delete_Flag, Created_By, Created_Date)
values(@AcademicYearId,@BranchId, @YearId, @SemesterId, @SectionId, @BatchId,@ElectiveBatch, CONVERT(DATETIME, @AttendanceDate,120), DATENAME(dw,CONVERT(DATETIME, @AttendanceDate,120)),'P', @Staff_id, @StudentTableId, 
@Course_id, @Period, @pRemarks, 1, 0, @pUserName, getdate())

END--Exists

FETCH NEXT FROM @CursorPeriod
INTO @Period

END--Period

CLOSE @CursorPeriod  
DEALLOCATE @CursorPeriod 

----------------------PERIOD LOOP END
FETCH NEXT FROM @CursorStudentId
INTO @StudentTableId

END--StudentId

CLOSE @CursorStudentId  
DEALLOCATE @CursorStudentId 

----------------------ATTENDANCE HISTORY START

IF(@pOdHdrId = 0)
BEGIN

DECLARE @CursorSection CURSOR
SET @CursorSection = CURSOR FAST_FORWARD
FOR
SELECT s FROM (Select s from SplitString(@plstSectionIds,','))T3

OPEN @CursorSection

FETCH NEXT FROM @CursorSection
INTO @strSectionId

WHILE @@FETCH_STATUS = 0
BEGIN

INSERT INTO T_ATTENDANCE_HISTRY (Staff_Id, Attendance_Date, Attendance_Day, Attendance_Type, Branch_Id, YearOfStudy_Id, Semester_Id, Section_Id,
Course_Id, Period, TopicsCovered, Created_By, Created_Date)
VALUES(@pUserId, CONVERT(DATETIME, @AttendanceDate,120), DATENAME(dw,CONVERT(DATETIME, @AttendanceDate,120)), 'Theoretical', 
@BranchId, @YearId, @SemesterId, CAST(@strSectionId AS BIGINT), 0, @plstPeriods, @pRemarks,
@pUserName, getdate())

FETCH NEXT FROM @CursorSection
INTO @strSectionId

END--Section

CLOSE @CursorSection  
DEALLOCATE @CursorSection 

END--Save History

----------------------ATTENDANCE HISTORY END

FETCH NEXT FROM @CursorDate
INTO @AttendanceDate

END--Date

CLOSE @CursorDate  
DEALLOCATE @CursorDate 


END--IsAttendance
----------------------SAVE ATTENDANCE END

END

--declare @a bigint
--exec sp_SaveOdAttendance '1613, 1614', '1,2', '2015-01-05 00:00:00.000, 2015-01-06 00:00:00.000', '2015-01-05 00:00:00.000', '2015-01-06 00:00:00.000', true, 'test', '1,2', 0,1, 'nagu', @a output
GO
/****** Object:  StoredProcedure [dbo].[sp_SelectedStudents_BranchWise]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[sp_SelectedStudents_BranchWise]
@AcademicYearId nvarchar(5),
@CompanyId nvarchar(5)
as
begin
select C.Company_Name 'Company Name'
, B.Branch_Code 'Branch Name'
, COUNT(case when (ISTU.IsAppeared = 1 and ISTU.InterviewStatus_Id = 5) then 1 else NULL end) 'Selected Students'
,COUNT(ISTU.Student_Id) 'EligibleStudents'
,CONVERT(DECIMAL(10,2),(COUNT(case when (ISTU.IsAppeared = 1 and ISTU.InterviewStatus_Id = 5) then 1 else NULL end)*100)/COUNT(ISTU.Student_Id)) 'Percentage'
from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,',')) 
join T_CFG_STUDENT_PERSONAL_DETAIL S on B.Branch_Id = S.Branch_Id
join T_INTERVIEW_STUDENTS ISTU on R.Id = ISTU.Recruiter_Id
where R.AcademicYear_Id = @AcademicYearId
and R.Company_Id = @CompanyId
and S.Id = ISTU.Student_Id
and r.Delete_Flag=0 and ISTU.Delete_Flag=0
group by B.Branch_Code,C.Company_Name
order by B.Branch_Code;


end

--sp_SelectedStudents_BranchWise 4,1
GO
/****** Object:  StoredProcedure [dbo].[sp_SelectedStudents_CompanyWise]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE procedure [dbo].[sp_SelectedStudents_CompanyWise]
@AcademicYearId nvarchar(5)
as
begin

select A.Academic_year_Name 'Academic Year', C.Company_Name 'Company Name'
,COUNT(case when (ISTU.IsAppeared = 1 and ISTU.InterviewStatus_Id = 5) then 1 else NULL end) 'Selected Students'
,COUNT(ISTU.Student_Id) 'EligibleStudents'
,CONVERT(DECIMAL(10,2),(COUNT(case when (ISTU.IsAppeared = 1 and ISTU.InterviewStatus_Id = 5) then 1 else NULL end)*100)/COUNT(ISTU.Student_Id)) 'Percentage'
from T_RECRUITER R
join T_CFG_COMPANY C on R.Company_Id = C.Id
join T_CFG_BRANCH B on B.Branch_Id in (select s from SplitString(R.Eligible_dept,',')) 
join T_CFG_STUDENT_PERSONAL_DETAIL S on B.Branch_Id = S.Branch_Id
join T_CFG_ACADEMIC_YEAR A on R.AcademicYear_Id = A.ID
join T_INTERVIEW_STUDENTS ISTU on R.Id = ISTU.Recruiter_Id
where R.AcademicYear_Id = @AcademicYearId
and S.Id = ISTU.Student_Id
and r.Delete_Flag=0 and ISTU.Delete_Flag=0
group by C.Company_Name,A.Academic_year_Name
order by C.Company_Name;


end

--sp_SelectedStudents_CompanyWise 4
GO
/****** Object:  StoredProcedure [dbo].[T_Proc_College_Strength]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
create procedure [dbo].[T_Proc_College_Strength]

as
Declare @vAcademicYr bigint;
set @vAcademicYr=(select ID from T_CFG_ACADEMIC_YEAR where flag=0 and Status=1 and Academic_year_Status=1)
select 
br.Branch_Id,br.Branch_Code,yr.YearOfStudy_Id,yr.YearOfStudy_Name,sem.Semester_Id,sem.Semester_Name
,count(case when stu.Gender like 'M%' Then 1 end) as Boys
,count(case when stu.Gender like 'F%' Then 1 end) as Girls
,count(stu.Id) as [Total Strength]
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_CFG_BRANCH br on stu.Branch_Id=br.Branch_Id
join T_CFG_YEAROFSTUDY_MASTER Yr on stu.YearOfStudy_Id = Yr.YearOfStudy_Id
join T_CFG_SEMESTER sem on stu.Semester_Id=sem.Semester_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id= sec.ID
where stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0 and stu.AcademicYear_Id=@vAcademicYr
--and stu.Branch_Id in (select id from CSVToTable(@plstBranchID))and stu.YearOfStudy_Id in (select id from CSVToTable(@plstYearId))
group by br.Branch_Id, br.Branch_Code,yr.YearOfStudy_Id,yr.YearOfStudy_Name,sem.Semester_Id,sem.Semester_Name
order by br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name



--exec T_Proc_College_Strength 
GO
/****** Object:  StoredProcedure [dbo].[T_Proc_College_Strength_SecWise]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE Proc [dbo].[T_Proc_College_Strength_SecWise]
as

Declare @vAcademicYr bigint;
set @vAcademicYr=(select ID from T_CFG_ACADEMIC_YEAR where flag=0 and Status=1 and Academic_year_Status=1)

select 
br.Branch_Id,br.Branch_Code,yr.YearOfStudy_Id,yr.YearOfStudy_Name,sem.Semester_Id,sem.Semester_Name,sec.ID Section_Id,sec.Section_Name
,count(case when stu.Gender like 'M%' Then 1 end) as Boys
,count(case when stu.Gender like 'F%' Then 1 end) as Girls
,count(stu.Id) as [Total Strength]
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_CFG_BRANCH br on stu.Branch_Id=br.Branch_Id
join T_CFG_YEAROFSTUDY_MASTER Yr on stu.YearOfStudy_Id = Yr.YearOfStudy_Id
join T_CFG_SEMESTER sem on stu.Semester_Id=sem.Semester_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id= sec.ID
where stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0 and stu.AcademicYear_Id=@vAcademicYr
--and stu.Branch_Id in (select id from CSVToTable(@plstBranchID))and stu.YearOfStudy_Id in (select id from CSVToTable(@plstYearId))
group by  br.Branch_Id, br.Branch_Code,yr.YearOfStudy_Id,yr.YearOfStudy_Name,sem.Semester_Id,sem.Semester_Name,sec.ID,sec.Section_Name
order by br.Branch_Code,yr.YearOfStudy_Name,sem.Semester_Name,sec.Section_Name






--exec T_Proc_College_Strength_SecWise
GO
/****** Object:  StoredProcedure [dbo].[test]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


CREATE Procedure [dbo].[test]

@pInsitutionId bigint, @pBranchId bigint, @pYearofStudyId bigint,@pAcademicYear bigint,@pFromDate nvarchar(20),@pToDate nvarchar(20)
As Begin
DECLARE @query1 AS NVARCHAR(MAX), @cols1 AS NVARCHAR(MAX)
;With StuInfo(StuTabId,StuName, StuUnivRegNo,StuBatchId,SemId)
As
(
select id,StudentName,UniversityRegNo,AcademicBatch_Id,Semester_Id
from T_CFG_STUDENT_PERSONAL_DETAIL
where AcademicYear_Id=@pAcademicYear and Institution_Id=@pInsitutionId and Branch_Id=@pBranchId and YearOfStudy_Id=@pYearofStudyId 
)
,
regulationInfo(regId)
As
(
select Regulation_Id from T_CFG_ACADEMICBATCH d where d.Id=(select Top 1 StuBatchId from StuInfo)
)
,

CourseInfo(SubId,SubCode,SubName)
As
(
select c.Course_Id,c.Course_Code,c.Course_Name from T_COURSE_REGULATION_SYALLBUS_HDR a
join T_COURSE_REGULATION_SYALLBUS_DTL b on a.ID = b.ID
join T_CFG_COURSE_MASTER c on b.course_code=c.Course_Id
where a.Regulation_ID=(select TOP 1 regId from regulationInfo)  and
	  a.Institution_ID=@pInsitutionId and a.Branch_Id=@pBranchId and a.YearOfStudy_Id=@pYearofStudyId and a.Semester_ID=(select top 1 SemId from StuInfo) and c.Course_Code not in ('','lab','-','PR','P and','Library')

	  
)
,

pivotInfo ([StuTabId],[Registration_Number],StuName,[CourseName],Present,[Absent])
As
(

select StuTabId as [StuTabId], StuUnivRegNo as [Registration Number], StuName as [StudentName],SubName as [CourseName],isnull(P,0) as Present,isnull(Ab,0) as [Absent]
from
(
select stu.StuTabId, stu.StuUnivRegNo, stu.StuName,CourseInfo.SubName,atndnce.Attendance_Status,count(atndnce.Attendance_Status) as AttendanceStatus from T_ATTENDANCEENTRY atndnce 
join StuInfo stu on atndnce.StudentTable_Id= stu.StuTabId
join CourseInfo on atndnce.Course_Id=CourseInfo.SubId
where atndnce.attendance_date between CONVERT(varchar(50),@pFromDate,103) And CONVERT(varchar(50),@pToDate,103) group by stu.StuName,CourseInfo.SubName,atndnce.Attendance_Status,stu.StuUnivRegNo,stu.StuTabId -- order by atndnce.Course_Id,atndnce.StudentTable_Id
) as source
pivot (max(AttendanceStatus) for Attendance_Status in([P],[Ab])) as pvt

)
--select @cols1=[CourseName] from pivotInfo
select  @cols1 = (STUFF((SELECT ',' + QUOTENAME(SubName) 
	FROM CourseInfo order by SubName FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)') ,1,1,''))

SET @query1 = 'select [Registration_Number] as [Register Number],StuName as [Name of student], '+@cols1+'

from
(
select [StuTabId],[Registration_Number],StuName,CourseName,(convert(nvarchar(5),Attended) + ''-'' + convert(nvarchar(5),Conducted) +''-'' +convert(nvarchar(10),[Attendance_Percentage])) as data
from
(select [StuTabId], [Registration_Number], StuName,CourseName,Present as Attended,[Absent] as Ab,
	   (Present+[Absent])as Conducted,
	   cast((Present*100)/((Present+[Absent])) as decimal(10,2))  as [Attendance_Percentage]
 from pivotInfo) as newDataTable
  --order by CourseName

) as newSource
 pivot(max(data) for CourseName in('+@cols1+')) as newPvt'

 execute(@query1)
end

--exec test 1,1,2,3,'2014-07-03','2014-07-23'



GO
/****** Object:  UserDefinedFunction [dbo].[CSVToTable]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO


CREATE FUNCTION [dbo].[CSVToTable] (@InStr VARCHAR(MAX))
RETURNS @TempTab TABLE
   (id int not null)
AS
BEGIN
    ;-- Ensure input ends with comma
	SET @InStr = REPLACE(@InStr + ',', ',,', ',')
	DECLARE @SP INT
DECLARE @VALUE VARCHAR(1000)
WHILE PATINDEX('%,%', @INSTR ) <> 0 
BEGIN
   SELECT  @SP = PATINDEX('%,%',@INSTR)
   SELECT  @VALUE = LEFT(@INSTR , @SP - 1)
   SELECT  @INSTR = STUFF(@INSTR, 1, @SP, '')
   INSERT INTO @TempTab(id) VALUES (@VALUE)
END
	RETURN
END


GO
/****** Object:  UserDefinedFunction [dbo].[fGetFailStudents]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE FUNCTION [dbo].[fGetFailStudents]
(
	-- Add the parameters for the function here
	@pProgmId bigint,
	@pYearId bigint,
	@pSemesterId bigint,
	@pExamId bigint
)
RETURNS 
@T TABLE 
(
	-- Add the column definitions for the TABLE variable here
	StudentId_Rollno nvarchar(50), 
	StudentTable_Id bigint,
	StudentName nvarchar(200),
	Branch_Code nvarchar(20),
	YearOfStudy_Name NVARCHAR(20),
	Section_Name NVARCHAR(20),
	FailCount int,
	Course varchar(Max)
)
AS
BEGIN
	Declare @vAcYr bigint,@vMinMark bigint
set @vAcYr=(SELECT ID FROM T_CFG_ACADEMIC_YEAR WHERE Status=1 AND flag=0 AND Academic_year_Status=1)
SET @vMinMark=(select settingValue from T_CFG_CONFIGURATON_SETTINGS where SettingName='PassMark' and IsActive=1 and IsDelete=0)

INSERT INTO @T

SELECT  S.StudentId_Rollno
,M.StudentTable_Id
,S.StudentName + ' ' + S.Initial AS StudentName
,b.Branch_Code
,YM.YearOfStudy_Name
,SM.Section_Name
,COUNT(M.StudentTable_Id) AS FailCount
,Course=Stuff((select ',' +convert(varchar(50),CM.Course_ShortCode) FROM T_MARK_ENTRY ME
		join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
		WHERE ME.AcademicYear_Id=@vAcYr AND ME.YearOfStudy_Id=@pYearId AND ME.Semester_Id=@pSemesterId AND ME.Exam_Id=@pExamId AND ME.Marks<@vMinMark AND M.StudentTable_Id=ME.StudentTable_Id
		GROUP BY ME.StudentTable_Id,CM.Course_ShortCode
		FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)'), 1, 1, '')
FROM T_MARK_ENTRY M
JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON M.StudentTable_Id = S.Id
JOIN T_CFG_BRANCH B ON s.BRANCH_ID = B.BRANCH_ID 
--JOIN T_CFG_COURSE_MASTER C ON M.COURSE_ID = C.COURSE_ID 
JOIN T_CFG_YEAROFSTUDY_MASTER YM ON YM.YearOfStudy_Id = s.YearOfStudy_Id 
--JOIN T_CFG_SEMESTER SEM ON s.Semester_Id = SEM.Semester_Id 
JOIN T_CFG_SECTION_MASTER SM ON SM.ID = s.Section_Id
WHERE m.AcademicYear_Id=@vAcYr AND M.YearOfStudy_Id=@pYearId AND M.Semester_Id=@pSemesterId AND M.Exam_Id=@pExamId AND M.Marks<@vMinMark
AND S.Delete_Flag=0 AND S.Status=1 AND S.TC_ISSUED=0 
AND B.Programme_Id=@pProgmId
GROUP BY M.StudentTable_Id,S.Institution_Id, S.StudentId_Rollno,S.StudentName,S.Initial,b.Branch_Code
,YM.YearOfStudy_Name,SM.Section_Name
	
	RETURN 
END


--select * from fGetFailStudents(1,2,3,1)
GO
/****** Object:  Table [dbo].[T_ACADEMIC_HISTORY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_ACADEMIC_HISTORY](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[STUDENT_ID] [bigint] NULL,
	[BRANCH_ID] [bigint] NULL,
	[YEAR_ID] [bigint] NULL,
	[SEMESTER_ID] [bigint] NULL,
	[SECTION_ID] [bigint] NULL,
	[ACADEMIC_ID] [bigint] NULL,
	[BATCH_ID] [bigint] NULL,
	[REMARKS] [nvarchar](max) NULL,
	[STATUS_FLAG] [bit] NULL,
	[DELETE_FLAG] [bit] NULL,
	[DATA_DATE] [datetime] NULL,
	[CREATED_BY] [nvarchar](50) NULL,
 CONSTRAINT [PK_T_ACADEMIC_HISTORY] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_ADMIN_ATTENDANCEENTRY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_ADMIN_ATTENDANCEENTRY](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Academic_Year] [bigint] NULL,
	[Branch_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[Staff_Id] [bigint] NULL,
	[StudentTable_Id] [bigint] NULL,
	[Course_Id] [bigint] NULL,
	[Attendance_Date] [datetime] NULL,
	[Attendance_Day] [nvarchar](50) NULL,
	[Attendance_Type] [nvarchar](50) NULL,
	[Period] [nvarchar](50) NULL,
	[PeriodType] [nvarchar](100) NULL,
	[Exam_Id] [bigint] NULL,
	[Attendance_Status] [nvarchar](50) NULL,
	[Batch_Id] [bigint] NULL,
	[Comments] [nvarchar](500) NULL,
	[TopicsCovered] [nvarchar](500) NULL,
	[AttendanceEdit_Remark] [nvarchar](500) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_ADMIN_AttendanceEntry] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_ASSIGNING_CLASS_DTL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_ASSIGNING_CLASS_DTL](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[Assigning_class_ID] [bigint] NOT NULL,
	[Section_Id] [bigint] NOT NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_date] [datetime] NULL,
	[FLAG] [bit] NULL,
	[STATUS] [bit] NULL,
 CONSTRAINT [PK_tbl_Assigning_class_Dtl] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_ASSIGNING_CLASS_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_ASSIGNING_CLASS_HDR](
	[Assigning_class_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[Academicyear_Id] [bigint] NOT NULL,
	[Institution_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NOT NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_date] [datetime] NULL,
	[Status] [bit] NOT NULL,
	[flag] [bit] NOT NULL,
 CONSTRAINT [PK_Assigning_class_Hdr] PRIMARY KEY CLUSTERED 
(
	[Assigning_class_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_ATTENDANCE_DELETE_HISTORY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_ATTENDANCE_DELETE_HISTORY](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[BRANCH_ID] [bigint] NULL,
	[YEAR_ID] [bigint] NULL,
	[SEMESTER_ID] [bigint] NULL,
	[SECTION_ID] [bigint] NULL,
	[ATTENDANCE_DATE] [date] NULL,
	[PERTIOD] [nvarchar](50) NULL,
	[DELETE_REMARKS] [nvarchar](500) NULL,
	[DELETE_FLAG] [bit] NULL,
	[CREATED_DATE] [datetime] NULL,
	[CREATED_BY] [nvarchar](200) NULL,
 CONSTRAINT [PK_T_ATTENDANCE_DELETE_HISTORY] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_ATTENDANCE_HISTRY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_ATTENDANCE_HISTRY](
	[AH_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Attendance_Date] [datetime] NULL,
	[Attendance_Day] [nvarchar](50) NULL,
	[Attendance_Type] [nvarchar](50) NULL,
	[Branch_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[Batch_Id] [bigint] NULL,
	[Staff_Id] [bigint] NULL,
	[Course_Id] [bigint] NULL,
	[Period] [nvarchar](50) NULL,
	[TopicsCovered] [nvarchar](500) NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
 CONSTRAINT [PK_Table_1] PRIMARY KEY CLUSTERED 
(
	[AH_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_ATTENDANCEENTRY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_ATTENDANCEENTRY](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Academic_Year] [bigint] NULL,
	[Branch_Id] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NOT NULL,
	[Section_Id] [bigint] NOT NULL,
	[Staff_Id] [bigint] NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[Course_Id] [bigint] NOT NULL,
	[Attendance_Date] [datetime] NULL,
	[Attendance_Day] [nvarchar](50) NULL,
	[Attendance_Type] [nvarchar](50) NULL,
	[Period] [nvarchar](50) NULL,
	[PeriodType] [nvarchar](100) NULL,
	[Exam_Id] [bigint] NULL,
	[Attendance_Status] [nvarchar](50) NULL,
	[Batch_Id] [bigint] NULL,
	[Elective_Batch_Id] [bigint] NULL,
	[Comments] [nvarchar](500) NULL,
	[TopicsCovered] [nvarchar](500) NULL,
	[AttendanceEdit_Remark] [nvarchar](500) NULL,
	[AttenRemarkId] [int] NULL,
	[Swapping_Remarks] [nvarchar](500) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_AttendanceEntry] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_ATTENDANCEENTRY_SCHOOL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_ATTENDANCEENTRY_SCHOOL](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Group_Id] [bigint] NOT NULL,
	[Institution_Id] [bigint] NOT NULL,
	[Staff_Id] [bigint] NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[Attendance_Date] [datetime] NULL,
	[Attendance_Type] [nvarchar](50) NULL,
	[Period] [nvarchar](10) NULL,
	[Attendance_Status] [nvarchar](20) NULL,
	[Comments] [nvarchar](150) NULL,
	[Status] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_AttendanceEntry_School] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_AU_REVIEW]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_AU_REVIEW](
	[Review_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Registration_No] [nvarchar](50) NULL,
	[Result_Type_Id] [bigint] NULL,
	[Result_Published_Year_Id] [bigint] NULL,
	[Review_Fees_Amount] [decimal](18, 2) NULL,
	[Review_Fees_Receipt_No] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_Review_Result] PRIMARY KEY CLUSTERED 
(
	[Review_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_AU_REVIEW_COURSES]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_AU_REVIEW_COURSES](
	[Review_CourseId] [bigint] IDENTITY(1,1) NOT NULL,
	[AU_Review_Id] [bigint] NULL,
	[AU_Result_Id] [bigint] NULL,
	[Course_Code] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
 CONSTRAINT [PK_tbl_AU_Review_Courses] PRIMARY KEY CLUSTERED 
(
	[Review_CourseId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_AUCOLLEGECODE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_AUCOLLEGECODE](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[CollegeName] [nvarchar](150) NULL,
	[CollegeCode] [nvarchar](50) NULL,
	[Range_From] [nvarchar](50) NULL,
	[Range_To] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_t_AUCollegeCode] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_AUEXAM_RESULTS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_AUEXAM_RESULTS](
	[Result_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Id] [bigint] NULL,
	[AUResult] [nvarchar](50) NULL,
	[Student_Registration_No] [nvarchar](50) NULL,
	[Student_Name] [nvarchar](150) NULL,
	[Branch_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Course_Id] [bigint] NULL,
	[Course_Code] [nvarchar](15) NULL,
	[Result_Type_Id] [bigint] NULL,
	[Grade] [nvarchar](10) NULL,
	[Internal_Mark] [int] NULL,
	[External_Mark] [int] NULL,
	[Result_Status] [nvarchar](50) NULL,
	[Result_Published_Year_Id] [bigint] NULL,
	[Is_Arrear_Paper] [bit] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_Exam_Results] PRIMARY KEY CLUSTERED 
(
	[Result_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_AURESULTSTUDENTDETAILS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_AURESULTSTUDENTDETAILS](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[RegistrationNumber] [nvarchar](50) NULL,
	[StudentName] [nvarchar](300) NULL,
	[Department] [nvarchar](300) NULL,
 CONSTRAINT [PK_TBL_AUResultStudentInfoFull_Jan2014] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_ACADEMIC_YEAR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_ACADEMIC_YEAR](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[Academic_year_code] [nvarchar](100) NOT NULL,
	[Academic_year_Name] [nvarchar](50) NOT NULL,
	[Academic_year_Status] [bit] NOT NULL,
	[Current_Semester] [nvarchar](50) NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_date] [datetime] NULL,
	[Status] [bit] NULL,
	[flag] [bit] NULL,
 CONSTRAINT [PK_T_CFG_ACADEMIC_YEAR] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_ACADEMICBATCH]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_ACADEMICBATCH](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Programme_Id] [bigint] NULL,
	[AcademicBatch_Code] [nvarchar](15) NULL,
	[AcademicBatch_Year] [nvarchar](15) NULL,
	[Regulation_Id] [bigint] NOT NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](150) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](150) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_AcademicBatch] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_ATTENDANCEREMARKS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_ATTENDANCEREMARKS](
	[ID] [int] IDENTITY(1,1) NOT NULL,
	[RemarksCode] [nvarchar](50) NULL,
	[RemarksDescription] [nvarchar](500) NULL,
 CONSTRAINT [PK_T_CFG_ATTENDANCEREMARKS] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_BRANCH]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_BRANCH](
	[Branch_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Id] [bigint] NULL,
	[Programme_Id] [bigint] NOT NULL,
	[Department_Id] [bigint] NOT NULL,
	[Branch_Code] [nvarchar](100) NOT NULL,
	[Branch_Name] [nvarchar](100) NULL,
	[Branch_ShortCode] [nvarchar](50) NULL,
	[Branch_Email] [nvarchar](500) NULL,
	[No_of_years] [int] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_Course] PRIMARY KEY CLUSTERED 
(
	[Branch_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_CERTIFICATE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_CERTIFICATE](
	[Certificate_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Certificate_Code] [nvarchar](15) NOT NULL,
	[Certificate_Name] [nvarchar](150) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_CertificateMaster] PRIMARY KEY CLUSTERED 
(
	[Certificate_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_COMPANY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_COMPANY](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Company_Name] [nvarchar](50) NULL,
	[Website] [nvarchar](100) NULL,
	[Company_LogoPath] [nvarchar](250) NULL,
	[UploadCompany_Presentation] [nvarchar](250) NULL,
	[Description] [nvarchar](500) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_T_CFG_COMPANY] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_CONFERENCE_TYPE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_CONFERENCE_TYPE](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[TYPE] [nvarchar](100) NULL,
	[STATUS] [bit] NULL,
	[DELETE_FLAG] [bit] NULL,
 CONSTRAINT [PK_T_CFG_CONFERENCE_TYPE] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_CONFIGURATON_SETTINGS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_CONFIGURATON_SETTINGS](
	[SettingId] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Id] [bigint] NULL,
	[SettingName] [nvarchar](100) NULL,
	[SettingValue] [nvarchar](100) NULL,
	[IsActive] [bit] NOT NULL,
	[IsDelete] [bit] NOT NULL,
 CONSTRAINT [PK_ConfigurationSetting] PRIMARY KEY CLUSTERED 
(
	[SettingId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_COUNTRY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CFG_COUNTRY](
	[Country_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Country_Code] [varchar](10) NULL,
	[Country_Name] [varchar](55) NULL,
	[Created_By] [varchar](15) NULL,
	[Data_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Country] PRIMARY KEY CLUSTERED 
(
	[Country_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_COURSE_MASTER]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_COURSE_MASTER](
	[Course_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Course_Code] [nvarchar](15) NOT NULL,
	[Course_ShortCode] [nvarchar](15) NULL,
	[Course_Name] [nvarchar](100) NULL,
	[Course_Type] [nvarchar](100) NULL,
	[TutorialType] [nvarchar](50) NULL,
	[Regulation_Id] [bigint] NULL,
	[LecturingHour] [int] NULL,
	[TutorialHour] [int] NULL,
	[Credit] [int] NULL,
	[practicleHour] [int] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[TextBook] [nvarchar](50) NULL,
	[ReferenceBook] [nvarchar](50) NULL,
	[SyllabusFilePath] [nvarchar](500) NULL,
	[No_of_Period] [nvarchar](50) NULL,
	[Is_Exam] [bit] NULL,
 CONSTRAINT [PK_T_CFG_COURSE_MASTER] PRIMARY KEY CLUSTERED 
(
	[Course_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_DEPARTMENT]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_DEPARTMENT](
	[Department_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Program_Id] [bigint] NULL,
	[Faculty_Id] [bigint] NOT NULL,
	[Department_Code] [nvarchar](15) NOT NULL,
	[Department_Name] [nvarchar](100) NULL,
	[NBA_Accreditation] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_cfg_Department] PRIMARY KEY CLUSTERED 
(
	[Department_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_DESIGNATION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_DESIGNATION](
	[Designation_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Designation_Code] [nvarchar](15) NOT NULL,
	[Designation_Name] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_Designation] PRIMARY KEY CLUSTERED 
(
	[Designation_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_DOCUMENT]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_DOCUMENT](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[DocumentCode] [nvarchar](50) NULL,
	[DocumentName] [nvarchar](150) NULL,
	[Description] [nvarchar](250) NULL,
	[Is_Common_Doc] [bit] NULL,
	[Status] [bit] NULL,
	[DeleteFlag] [bit] NULL,
 CONSTRAINT [PK_tbl_UniversityDocument] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_ELECTIVECOURSE_STAFF_PERIODS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_ELECTIVECOURSE_STAFF_PERIODS](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Academic_Year_Id] [bigint] NULL,
	[Branch_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[Staff_Id] [bigint] NULL,
	[Course_Id] [bigint] NULL,
	[DayName] [nvarchar](50) NULL,
	[Periods] [nvarchar](50) NULL,
	[ElectiveBatch_Id] [bigint] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
 CONSTRAINT [PK_T_CFG_ELECTIVECOURSE_STAFF_PERIODS] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_ERRORLOG]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_ERRORLOG](
	[Error_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[ProjectName] [nvarchar](100) NULL,
	[FormName] [nvarchar](100) NULL,
	[ProcessName] [nvarchar](100) NULL,
	[Severity] [nvarchar](50) NULL,
	[ErrorDetails] [nvarchar](max) NULL,
	[UserName] [nvarchar](50) NULL,
	[RecordStatus] [nvarchar](50) NULL,
	[Data_Date] [datetime] NULL,
	[IsDelete] [bit] NULL,
 CONSTRAINT [PK_ErrorLog] PRIMARY KEY CLUSTERED 
(
	[Error_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_EXAM_MASTER]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_EXAM_MASTER](
	[Exam_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Exam_Code] [nvarchar](15) NOT NULL,
	[Exam_Type] [nvarchar](50) NULL,
	[Exam_Name] [nvarchar](100) NULL,
	[ClassAfterTest] [bit] NULL,
	[Is_University_Exam] [bit] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Retest_Status] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Order_By] [int] NULL,
 CONSTRAINT [PK_tbl_cfg_Exam_Master] PRIMARY KEY CLUSTERED 
(
	[Exam_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_FACULTY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_FACULTY](
	[Faculty_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Faculty_Code] [nvarchar](15) NOT NULL,
	[Faculty_Name] [nvarchar](150) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_Faculty] PRIMARY KEY CLUSTERED 
(
	[Faculty_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_FEES_EXECMPTION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_FEES_EXECMPTION](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[CGPA_Range] [nvarchar](50) NULL,
	[Amount] [decimal](18, 0) NULL,
 CONSTRAINT [PK_tbl_cfg_feesexcemption] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_FEES_EXECMPTION_MASTER]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_FEES_EXECMPTION_MASTER](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[CGPA_Range] [nvarchar](50) NULL,
	[Amount] [decimal](18, 0) NULL,
 CONSTRAINT [PK_T_CFG_FEES_EXECMPTION_MASTER] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_FEES_TYPE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_FEES_TYPE](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[TYPE] [nvarchar](100) NOT NULL,
	[STATUS] [bit] NULL,
	[DELETE_FLAG] [bit] NULL,
 CONSTRAINT [PK_T_CFG_FEES_TYPE] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_GRADE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_GRADE](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[Regulation_Id] [bigint] NULL,
	[Letter_Grade] [nvarchar](50) NULL,
	[Grade_Points] [int] NULL,
	[Marks_Range] [nvarchar](50) NULL,
	[Delete_Flag] [bit] NULL,
	[IsPass] [bit] NULL,
 CONSTRAINT [PK_T_CFG_GRADE] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_HALLTYPE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CFG_HALLTYPE](
	[Hall_Id] [bigint] NOT NULL,
	[HallType] [varchar](50) NULL,
 CONSTRAINT [PK_t_HallType] PRIMARY KEY CLUSTERED 
(
	[Hall_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_HOLIDAY_DECLARATION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_HOLIDAY_DECLARATION](
	[Holiday_Id] [int] IDENTITY(1,1) NOT NULL,
	[Start_Date] [nvarchar](500) NULL,
	[End_Date] [nvarchar](500) NULL,
	[Title] [nvarchar](50) NULL,
	[All_Day] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_cfg_HolidayDeclaration_1] PRIMARY KEY CLUSTERED 
(
	[Holiday_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_INSTITUTION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_INSTITUTION](
	[Institution_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Group_Id] [bigint] NOT NULL,
	[Stateid] [bigint] NOT NULL,
	[Country_Id] [bigint] NULL,
	[Institution_Code] [nvarchar](50) NULL,
	[Institution_Name] [nvarchar](200) NULL,
	[Institute_Type_Id] [bigint] NULL,
	[Address] [nvarchar](500) NULL,
	[Postalcode] [nvarchar](10) NULL,
	[City] [nvarchar](100) NULL,
	[WebSite] [nvarchar](500) NULL,
	[EmailId] [nvarchar](500) NULL,
	[Contact_No] [nvarchar](100) NULL,
	[Status] [bit] NULL,
	[Approved_By] [nvarchar](100) NULL,
	[Affiliated_With] [nvarchar](100) NULL,
	[NBA_Accreditation] [nvarchar](50) NULL,
	[ISO_Certification] [nvarchar](50) NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
	[Institution_Logo] [nvarchar](500) NULL,
	[Institution_WaterImage] [nvarchar](500) NULL,
 CONSTRAINT [PK_tbl_cfg_Institution] PRIMARY KEY CLUSTERED 
(
	[Institution_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_INSTITUTION_BRANCH]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_INSTITUTION_BRANCH](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_Institution_Course] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_INSTITUTION_GROUP]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_INSTITUTION_GROUP](
	[Institution_Group_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Country_Id] [bigint] NULL,
	[Stateid] [bigint] NOT NULL,
	[Institution_Group_Code] [nvarchar](50) NULL,
	[Institution_Group_Name] [nvarchar](200) NULL,
	[Institution_Type_Id] [bigint] NULL,
	[Address] [nvarchar](500) NULL,
	[Postalcode] [nvarchar](10) NULL,
	[City] [nvarchar](100) NULL,
	[Website] [nvarchar](500) NULL,
	[EmailId] [nvarchar](200) NULL,
	[Contact_No] [nvarchar](100) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
 CONSTRAINT [PK_tbl_cfg_Institution_Group] PRIMARY KEY CLUSTERED 
(
	[Institution_Group_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_INSTITUTION_TYPE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_INSTITUTION_TYPE](
	[id] [bigint] IDENTITY(1,1) NOT NULL,
	[InstitutionType] [nvarchar](150) NULL,
 CONSTRAINT [PK_tbl_cfg_InstitutionType] PRIMARY KEY CLUSTERED 
(
	[id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_LAB_BATCH]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_LAB_BATCH](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[LabBatch_No] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
 CONSTRAINT [PK_tbl_cfg_LabBatch] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_LAB_BLACKS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_LAB_BLACKS](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[LabBlackCode] [nvarchar](50) NULL,
	[LabBlackName] [nvarchar](50) NULL,
	[Description] [nvarchar](500) NULL,
	[Status] [bit] NOT NULL,
	[Delete_Flag] [bit] NOT NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_LabBlocks] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_LAB_TIMETABLE_DEFINITION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_LAB_TIMETABLE_DEFINITION](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NOT NULL,
	[Section_Id] [bigint] NOT NULL,
	[Student_Batch_Id] [bigint] NOT NULL,
	[Lab_Course_Id] [bigint] NOT NULL,
	[Lab_Branch_Id] [bigint] NOT NULL,
	[Lab_Black_ID] [bigint] NOT NULL,
	[Staff_Id] [bigint] NOT NULL,
	[Day] [nvarchar](50) NULL,
	[Periods] [nvarchar](50) NULL,
	[Status] [bit] NOT NULL,
	[Delete_Flag] [bit] NOT NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_Lab_TimeTable_Definition] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_LEAVE_REASON]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_LEAVE_REASON](
	[LeaveReasonId] [bigint] IDENTITY(1,1) NOT NULL,
	[ReasonName] [nvarchar](100) NULL,
	[Order_By] [int] NULL,
	[Status] [bit] NULL,
	[IsDeleted] [bit] NULL,
	[CreatedBy] [nvarchar](100) NULL,
	[CreatedDate] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_LeaveReason] PRIMARY KEY CLUSTERED 
(
	[LeaveReasonId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_MODULE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CFG_MODULE](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[ModuleIcon] [nvarchar](50) NULL,
	[ModuleName] [varchar](50) NULL,
	[ModuleDesc] [varchar](500) NULL,
	[OrderBy] [int] NULL,
	[IsDeleted] [bit] NULL,
	[CreatedBy] [int] NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [int] NULL,
	[ModifiedDate] [datetime] NULL,
 CONSTRAINT [PK_Module] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_MODULEPERMISSION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_MODULEPERMISSION](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[RoleId] [int] NOT NULL,
	[ModuleId] [int] NOT NULL,
	[IsDeleted] [bit] NULL,
	[CreatedBy] [int] NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [int] NULL,
	[ModifiedDate] [datetime] NULL,
 CONSTRAINT [PK_ModulePermission] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_MODULETOSCREENS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_MODULETOSCREENS](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[ScreenId] [int] NOT NULL,
	[ModuleId] [int] NOT NULL,
	[SubModuleId] [int] NOT NULL,
	[OrderBy] [int] NULL,
	[IsDeleted] [bit] NULL,
	[CreatedBy] [int] NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [int] NULL,
	[ModifiedDate] [datetime] NULL,
 CONSTRAINT [PK_ModuleToScreens] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_MULTIPLE_STAFF_PERIODS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_MULTIPLE_STAFF_PERIODS](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Academic_Year_Id] [bigint] NULL,
	[Branch_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[Staff_Id] [bigint] NULL,
	[Course_Id] [bigint] NULL,
	[DayName] [nvarchar](50) NULL,
	[Periods] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
 CONSTRAINT [PK_T_CFG_MULTIPLE_STAFF_PERIODS] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_ORDER_BY_DETAILS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_ORDER_BY_DETAILS](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[ORDER_BY_VALUE] [nvarchar](50) NULL,
	[ORDER_BY_NAME] [nvarchar](100) NULL,
 CONSTRAINT [PK_T_CFG_ORDER_BY_DETAILS] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_ORGANIZATION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CFG_ORGANIZATION](
	[Org_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Org_Code] [nvarchar](15) NULL,
	[Org_Name] [nvarchar](200) NULL,
	[Org_Address] [nvarchar](500) NULL,
	[Org_City] [nvarchar](50) NULL,
	[Org_Pincode] [nvarchar](15) NULL,
	[Org_CountryId] [bigint] NULL,
	[Org_StateId] [bigint] NULL,
	[Org_Contactno] [varchar](50) NULL,
	[Email] [varchar](50) NULL,
	[website] [varchar](50) NULL,
	[C_P_Coordinator] [varchar](50) NULL,
	[C_P_C_designation] [varchar](50) NULL,
	[C_P_C_Mobileno] [varchar](50) NULL,
	[Org_Logoname] [nvarchar](max) NULL,
	[Org_Logopath] [nvarchar](max) NULL,
	[Order_By] [int] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_Organisation] PRIMARY KEY CLUSTERED 
(
	[Org_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_PERIOD]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_PERIOD](
	[PeriodId] [int] IDENTITY(1,1) NOT NULL,
	[Period] [nvarchar](100) NULL,
	[Order] [int] NULL,
	[CreatedBy] [nvarchar](100) NULL,
	[Data_Date] [datetime] NULL,
	[IsActive] [bit] NULL,
	[Flag] [bit] NULL,
 CONSTRAINT [PK_tbl_Period] PRIMARY KEY CLUSTERED 
(
	[PeriodId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_PLACEMENT_TRAINERS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_PLACEMENT_TRAINERS](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Trainer_Code] [nvarchar](15) NULL,
	[Trainer_Name] [nvarchar](150) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_T_CFG_PLACEMENT_TRAINERS] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_PRIVILEGE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING OFF
GO
CREATE TABLE [dbo].[T_CFG_PRIVILEGE](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[PrivilegeName] [varchar](50) NULL,
	[PrivilegeDesc] [varchar](500) NULL,
	[IsDeleted] [bit] NULL,
	[CreatedBy] [int] NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [int] NULL,
	[ModifiedDate] [datetime] NULL,
	[OrderBy] [int] NULL,
 CONSTRAINT [PK_Privilege] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_PROGRAMME]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_PROGRAMME](
	[Programme_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Programme_Code] [nvarchar](15) NOT NULL,
	[Programme_Name] [nvarchar](100) NULL,
	[Programme_Type] [nvarchar](10) NULL,
	[Programme_NoOfYears] [int] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_Degree] PRIMARY KEY CLUSTERED 
(
	[Programme_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_PROJECT]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_PROJECT](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[ProjectType_Id] [bigint] NULL,
	[InstitutionID] [bigint] NULL,
	[BranchID] [bigint] NULL,
	[YearIdD] [bigint] NULL,
	[SemesterID] [bigint] NULL,
	[SectionID] [bigint] NULL,
	[StudentTableID] [bigint] NULL,
	[ProjectTitle] [nvarchar](200) NULL,
	[ProjectDescription] [nvarchar](max) NULL,
	[ProjectGuideId] [bigint] NULL,
	[Project_Review] [nvarchar](500) NULL,
	[DeleteFlag] [bit] NULL,
	[CRAETED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_t_Project] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_PROJECT_TYPE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_PROJECT_TYPE](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[TYPE] [nvarchar](100) NULL,
	[STATUS] [bit] NULL,
	[DELETE_FLAG] [bit] NULL,
 CONSTRAINT [PK_T_CFG_PROJECT_TYPE] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_Recipients]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_Recipients](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Email_Type] [nvarchar](100) NULL,
	[Recipient_Type] [nvarchar](100) NULL,
	[FromEmail] [nvarchar](200) NULL,
	[ToEmails] [nvarchar](max) NULL,
	[BCCs] [nvarchar](max) NULL,
	[CCs] [nvarchar](max) NULL,
	[Subject] [nvarchar](max) NULL,
	[Signature] [nvarchar](max) NULL,
	[AlertBeforeDay_Satff] [int] NULL,
	[AlertBeforeDay_HOD] [int] NULL,
	[AlertBeforeDay_Admin] [int] NULL,
	[Active] [bit] NULL,
	[Delete_Flag] [bit] NULL,
 CONSTRAINT [PK_T_CFG_Recipients] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_REGULATION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_REGULATION](
	[Regulation_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Regulation_Code] [nvarchar](15) NOT NULL,
	[Regulation_Year] [nvarchar](10) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [int] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_Regulation] PRIMARY KEY CLUSTERED 
(
	[Regulation_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_REPORTS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CFG_REPORTS](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Reprot_Name] [varchar](100) NULL,
	[Description] [nvarchar](250) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
 CONSTRAINT [PK_T_CFG_REPORTS] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_RESPONSE_QUERY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_RESPONSE_QUERY](
	[RESPONSE_QUERY_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[INSTITUTION_ID] [bigint] NULL,
	[RAISE_QUERY_ID] [bigint] NOT NULL,
	[RESPONSE_BY] [bigint] NOT NULL,
	[RESPONSE_STATUS] [nvarchar](50) NULL,
	[DESCRIPTION] [nvarchar](max) NOT NULL,
	[FLAG] [bit] NULL,
	[STATUS] [bit] NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_tbl_Response_Query] PRIMARY KEY CLUSTERED 
(
	[RESPONSE_QUERY_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_RESULT_PUBLISH_YEAR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_RESULT_PUBLISH_YEAR](
	[Result_PublishYear_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[PublishYear] [nvarchar](500) NULL,
	[Semester_Id] [bigint] NULL,
	[DateOfResultPublished] [datetime] NULL,
	[YearOfPass] [nvarchar](100) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_Result_PublishYear] PRIMARY KEY CLUSTERED 
(
	[Result_PublishYear_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_RESULTTYPE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_RESULTTYPE](
	[Result_Type_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Result_TypeName] [nvarchar](500) NULL,
	[Result_Type] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_Result_Types] PRIMARY KEY CLUSTERED 
(
	[Result_Type_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_ROLES]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING OFF
GO
CREATE TABLE [dbo].[T_CFG_ROLES](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[RoleName] [varchar](50) NULL,
	[RoleDesc] [varchar](500) NULL,
	[IsDeleted] [bit] NULL,
	[CreatedBy] [int] NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [int] NULL,
	[ModifiedDate] [datetime] NULL,
	[OrderBy] [int] NULL,
 CONSTRAINT [PK_Roles] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_ROLESPERMISSION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_ROLESPERMISSION](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[RoleId] [int] NOT NULL,
	[PrivilegeId] [int] NULL,
	[ModuleId] [int] NOT NULL,
	[SubModuleId] [int] NOT NULL,
	[ScreenId] [int] NOT NULL,
	[bCreate] [bit] NULL,
	[bView] [bit] NULL,
	[bUpdate] [bit] NULL,
	[bDelete] [bit] NULL,
	[IsDeleted] [bit] NULL,
	[CreatedBy] [int] NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [int] NULL,
	[ModifiedDate] [datetime] NULL,
	[OrderBy] [int] NULL,
 CONSTRAINT [PK_RolesPermission] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_SATURDAY_TIMETABLE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_SATURDAY_TIMETABLE](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[AcademicYear_Id] [bigint] NULL,
	[Saturday_Attendance_Date] [datetime] NULL,
	[Attendance_Day] [nvarchar](50) NULL,
	[ProgrameId] [bigint] NULL,
	[Years] [nvarchar](50) NULL,
	[Remarks] [nvarchar](500) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Created_Date] [datetime] NULL,
 CONSTRAINT [PK_T_CFG_SATURDAY_TIMETABLE] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_SCHOLARSHIP_AMOUNT]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_SCHOLARSHIP_AMOUNT](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[CGPA_Range] [nvarchar](50) NULL,
	[Amount] [decimal](18, 0) NULL,
 CONSTRAINT [PK_tbl_cfg_ScholarshipAmount] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_SCREEN]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CFG_SCREEN](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[ScreenIcon] [nvarchar](50) NULL,
	[ScreenName] [varchar](50) NULL,
	[ScreenDesc] [varchar](500) NULL,
	[Action] [varchar](50) NULL,
	[Controller] [varchar](50) NULL,
	[IsDeleted] [bit] NULL,
	[OrderBy] [int] NULL,
	[CreatedBy] [int] NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [int] NULL,
	[ModifiedDate] [datetime] NULL,
 CONSTRAINT [PK_Screen] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_SECTION_MASTER]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_SECTION_MASTER](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[Section_code] [nvarchar](15) NOT NULL,
	[Section_Name] [nvarchar](50) NOT NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Status] [bit] NOT NULL,
	[Flag] [bit] NULL,
 CONSTRAINT [PK_tbl_section_master] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_SEMESTER]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_SEMESTER](
	[Semester_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[Semester_Code] [nvarchar](15) NULL,
	[Semester_Name] [nvarchar](50) NULL,
	[Semester_Type] [nvarchar](50) NULL,
	[Active_Semester] [bit] NULL,
	[Start_Month] [nvarchar](50) NULL,
	[End_Month] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_Semester] PRIMARY KEY CLUSTERED 
(
	[Semester_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_SEMESTER_DATE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_SEMESTER_DATE](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[AcademicYear_Id] [bigint] NULL,
	[Programme_Id] [bigint] NULL,
	[Branch_Ids] [nvarchar](50) NULL,
	[Semester_Ids] [nvarchar](50) NULL,
	[FromDate] [datetime] NULL,
	[ToDate] [datetime] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_T_CFG_SEMESTER_DATE] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_STAFF_LABPERIODS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_STAFF_LABPERIODS](
	[StaffLab_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Academic_Year_Id] [bigint] NULL,
	[Branch_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[Staff_Id] [bigint] NULL,
	[Course_Id] [bigint] NULL,
	[DayName] [nvarchar](50) NULL,
	[LabPeriods] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
 CONSTRAINT [PK_T_CFG_STAFF_LABPERIODS] PRIMARY KEY CLUSTERED 
(
	[StaffLab_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_STAFF_MASTER]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_STAFF_MASTER](
	[Staff_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Staff_Code] [nvarchar](15) NULL,
	[Title] [nvarchar](50) NULL,
	[Staff_Name] [nvarchar](100) NULL,
	[Institution_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NULL,
	[Designation_Id] [bigint] NOT NULL,
	[Staff_Catogary] [bigint] NULL,
	[Pay_Scale] [nvarchar](500) NULL,
	[Date_Of_Promotion] [datetime] NULL,
	[Gender] [nvarchar](6) NULL,
	[Date_Of_Birth] [datetime] NULL,
	[Father_Or_Husband_Name] [nvarchar](100) NULL,
	[Permanant_Address] [nvarchar](500) NULL,
	[Permenant_City] [nvarchar](50) NULL,
	[Permenant_Pincode] [nvarchar](10) NULL,
	[Permenant_State_id] [bigint] NULL,
	[Permenant_Country_Id] [bigint] NULL,
	[Permanent_Landline_No] [nvarchar](15) NULL,
	[Permanent_Mobile_No] [nvarchar](15) NULL,
	[Email_Personal] [nvarchar](100) NULL,
	[Email_Official] [nvarchar](100) NULL,
	[Present_Address] [nvarchar](500) NULL,
	[Present_City] [nvarchar](50) NULL,
	[Present_Pincode] [nvarchar](6) NULL,
	[Prsent_State_id] [bigint] NULL,
	[Present_Country_Id] [bigint] NULL,
	[Present_Landline_No] [nvarchar](15) NULL,
	[Present_Mobile_No] [nvarchar](15) NULL,
	[Date_Of_Joining] [datetime] NULL,
	[Date_Of_Resignation] [datetime] NULL,
	[Date_Of_Leaving] [datetime] NULL,
	[Reason_For_Leaving] [nvarchar](500) NULL,
	[Staff_Type] [nvarchar](20) NULL,
	[PanCard_Number] [nvarchar](50) NULL,
	[Aadhar_UID] [nvarchar](50) NULL,
	[VoterId_Number] [nvarchar](50) NULL,
	[Bank_Account_Number] [nvarchar](50) NULL,
	[Loans_From_Management] [decimal](18, 0) NULL,
	[Loan_Date] [datetime] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Deleted_Remarks] [nvarchar](500) NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Staff_Image] [image] NULL,
	[Flag_Update] [bit] NULL,
 CONSTRAINT [PK_tbl_cfg_Employee_Master] PRIMARY KEY CLUSTERED 
(
	[Staff_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_STATES]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CFG_STATES](
	[Stateid] [bigint] IDENTITY(1,1) NOT NULL,
	[Country_Id] [bigint] NULL,
	[StateName] [varchar](100) NOT NULL,
 CONSTRAINT [PK_tbl_cfg_States] PRIMARY KEY CLUSTERED 
(
	[Stateid] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_STUDENT_ACADEMIC_DETAIL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_STUDENT_ACADEMIC_DETAIL](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[Qualifying_Exam] [nvarchar](100) NULL,
	[CutOffMarks_Scored] [decimal](18, 2) NULL,
	[CutOffMarks_Max] [decimal](18, 2) NULL,
	[Mathematics_Scored] [decimal](18, 2) NULL,
	[Mathematics_Max] [decimal](18, 2) NULL,
	[Physics_Scored] [decimal](18, 2) NULL,
	[Physics_Max] [decimal](18, 2) NULL,
	[Chemistry_Scored] [decimal](18, 2) NULL,
	[Chemistry_Max] [decimal](18, 2) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
	[X_Reg_No] [nvarchar](20) NULL,
	[X_Institution_Name] [nvarchar](250) NULL,
	[X_Board] [nvarchar](50) NULL,
	[X_YearOfPassing] [nvarchar](4) NULL,
	[X_Group] [nvarchar](50) NULL,
	[X_Discilpline] [nvarchar](50) NULL,
	[X_Medium] [nvarchar](50) NULL,
	[X_Mark_Obtained] [decimal](18, 0) NULL,
	[X_Max_Marks] [decimal](18, 0) NULL,
	[X_Percentage] [decimal](18, 0) NULL,
	[XII_Reg_No] [nvarchar](20) NULL,
	[XII_Institution_Name] [nvarchar](250) NULL,
	[XII_Board] [nvarchar](50) NULL,
	[XII_YearOfPassing] [nvarchar](8) NULL,
	[XII_Group] [nvarchar](50) NULL,
	[XII_Discilpline] [nvarchar](50) NULL,
	[XII_Medium] [nvarchar](50) NULL,
	[XII_AddressOfSchool] [nvarchar](250) NULL,
	[XII_DistrictOfSchool] [nvarchar](50) NULL,
	[XII_ExaminationAppeared] [nvarchar](50) NULL,
	[XII_Mark_Obtained] [decimal](18, 0) NULL,
	[XII_Max_Marks] [decimal](18, 0) NULL,
	[XII_Percentage] [decimal](18, 0) NULL,
	[UG_Degree_Id] [bigint] NULL,
	[UG_Degree_Reg_No] [nvarchar](20) NULL,
	[UG_Degree_Discipline] [nvarchar](150) NULL,
	[UG_Degree_Institution_Name] [nvarchar](150) NULL,
	[UG_Degree_Institution_Address] [nvarchar](150) NULL,
	[UG_Degree_University_Board] [nvarchar](150) NULL,
	[UG_Degree_YearOfPassing] [nvarchar](4) NULL,
	[UG_Degree_Study_Mode] [nvarchar](25) NULL,
	[UG_Medium] [nvarchar](50) NULL,
	[UG_Degree_State] [nvarchar](50) NULL,
	[UG_Percentage] [decimal](18, 0) NULL,
	[UG_Grade] [nvarchar](10) NULL,
	[UG_Pattern] [nvarchar](100) NULL,
	[UG_Class] [nvarchar](50) NULL,
	[UG_Sem1] [decimal](18, 0) NULL,
	[UG_Sem2] [decimal](18, 0) NULL,
	[UG_Sem3] [decimal](18, 0) NULL,
	[UG_Sem4] [decimal](18, 0) NULL,
	[UG_Sem5] [decimal](18, 0) NULL,
	[UG_Sem6] [decimal](18, 0) NULL,
	[UG_Sem7] [decimal](18, 0) NULL,
	[UG_Sem8] [decimal](18, 0) NULL,
	[UG_Total] [decimal](18, 0) NULL,
	[UG_TotalMax] [decimal](18, 0) NULL,
	[UG_Cgpa] [decimal](18, 0) NULL,
	[UG_CgpaMax] [decimal](18, 0) NULL,
	[PG_Degree_Id] [bigint] NULL,
	[PG_Degree_Reg_No] [nvarchar](20) NULL,
	[PG_Degree_Institution_Name] [nvarchar](250) NULL,
	[PG_Degree_University_Board] [nvarchar](50) NULL,
	[PG_Degree_YearOfPassing] [nvarchar](4) NULL,
	[PG_Degree_Study_Mode] [nvarchar](50) NULL,
	[PG_Pattern_of_Study] [nvarchar](100) NULL,
	[PG_Degree_State] [nvarchar](50) NULL,
	[PG_Percentage] [decimal](18, 0) NULL,
	[PG_Degree_Discipline] [nvarchar](150) NULL,
	[PG_Grade] [nvarchar](10) NULL,
	[PG_Pattern] [nvarchar](150) NULL,
	[PG_Class] [nvarchar](25) NULL,
	[PG_Sem1] [decimal](18, 0) NULL,
	[PG_Sem2] [decimal](18, 0) NULL,
	[PG_Sem3] [decimal](18, 0) NULL,
	[PG_Sem4] [decimal](18, 0) NULL,
	[PG_Sem5] [decimal](18, 0) NULL,
	[PG_Sem6] [decimal](18, 0) NULL,
	[PG_Sem7] [decimal](18, 0) NULL,
	[PG_Sem8] [decimal](18, 0) NULL,
	[PG_Total] [decimal](18, 0) NULL,
	[PG_TotalMax] [decimal](18, 0) NULL,
	[PG_Cgpa] [decimal](18, 0) NULL,
	[PG_CgpaMax] [decimal](18, 0) NULL,
	[PG_Entrance_Exam_Name] [nvarchar](500) NULL,
	[PG_Entrance_Exam_RegNo] [nvarchar](50) NULL,
	[PG_Entrance_Exam_Mark] [decimal](18, 0) NULL,
	[MathsStudiedLevel] [nvarchar](150) NULL,
	[MathsStudiedMca] [nvarchar](50) NULL,
	[IsWorkingExperience] [nvarchar](50) NULL,
	[YearofExperience] [nvarchar](15) NULL,
	[ExperienceDescription] [nvarchar](500) NULL,
	[AIEE] [nvarchar](250) NULL,
 CONSTRAINT [PK_tbl_cfg_StudentAcademicInfo] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_STUDENT_FAMILY_DETAIL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_STUDENT_FAMILY_DETAIL](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[Relationship] [nvarchar](100) NULL,
	[Name] [nvarchar](150) NULL,
	[Occupation] [nvarchar](150) NULL,
	[StudiedIn_SameGroup] [nvarchar](50) NULL,
	[Course_OfStudy] [nvarchar](150) NULL,
	[Year] [nvarchar](50) NULL,
	[Occupation_Designnation] [nvarchar](150) NULL,
	[Address_OfOffice] [nvarchar](max) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](150) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](150) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_StudentFamilyInfo] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_STUDENT_OTHER_DETAIL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_STUDENT_OTHER_DETAIL](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[MoveFree_InClass] [nvarchar](max) NULL,
	[Personal_Problems] [nvarchar](max) NULL,
	[Health_Condition] [nvarchar](max) NULL,
	[Undergoing_MedicalTreatment] [nvarchar](15) NULL,
	[If_MedicalTreatment] [nvarchar](max) NULL,
	[OtherAreas_OfInterest] [nvarchar](max) NULL,
	[Hobbies] [nvarchar](max) NULL,
	[NCC] [nvarchar](200) NULL,
	[Special_Achievements] [nvarchar](max) NULL,
	[Sport_Interest] [nvarchar](max) NULL,
	[Talent] [nvarchar](max) NULL,
	[Ambition] [nvarchar](max) NULL,
	[Reason] [nvarchar](max) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](150) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](150) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_StudentOtherInfo] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_STUDENT_PARENT_DETAIL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_STUDENT_PARENT_DETAIL](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[F_Status] [nvarchar](50) NULL,
	[F_Name] [nvarchar](150) NULL,
	[F_Occupation] [nvarchar](100) NULL,
	[F_Office_Address] [nvarchar](max) NULL,
	[F_Mobile_Number] [nvarchar](15) NULL,
	[F_LandLineNumber] [nvarchar](25) NULL,
	[F_Email_Id] [nvarchar](150) NULL,
	[F_Photo] [bit] NULL,
	[F_PhotoPath] [nvarchar](1000) NULL,
	[M_Name] [nvarchar](150) NULL,
	[M_Occupation] [nvarchar](100) NULL,
	[M_Office_Address] [nvarchar](max) NULL,
	[M_Mobile_Number] [nvarchar](15) NULL,
	[M_LandLineNumber] [nvarchar](25) NULL,
	[M_Email_Id] [nvarchar](150) NULL,
	[M_Photo] [bit] NULL,
	[M_PhotoPath] [nvarchar](1000) NULL,
	[G_Name] [nvarchar](150) NULL,
	[G_Occupation] [nvarchar](100) NULL,
	[G_Office_Address] [nvarchar](max) NULL,
	[G_Mobile_Number] [nvarchar](15) NULL,
	[G_LandLineNumber] [nvarchar](25) NULL,
	[G_Email_Id] [nvarchar](150) NULL,
	[G_Photo] [bit] NULL,
	[G_PhotoPath] [nvarchar](1000) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](150) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](150) NULL,
	[Modified_Date] [datetime] NULL,
	[F_AnnualIncome] [nvarchar](100) NULL,
	[G_RelationShipWithStudent] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_cfg_StudentGuardianInfo] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_STUDENT_PERSONAL_DETAIL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CFG_STUDENT_PERSONAL_DETAIL](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Application_Number] [nvarchar](20) NULL,
	[SNo] [bigint] NOT NULL,
	[Institution_Id] [bigint] NOT NULL,
	[StudentId_Rollno] [nvarchar](100) NULL,
	[StudentName] [nvarchar](100) NULL,
	[Initial] [nvarchar](100) NULL,
	[UniversityRegNo] [nvarchar](100) NULL,
	[AcademicBatch_Id] [bigint] NULL,
	[AcademicYear_Id] [bigint] NULL,
	[Branch_Id] [bigint] NOT NULL,
	[Programme_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[Joining_Type] [nvarchar](10) NULL,
	[Is_Hostel] [bit] NULL,
	[LabBatch_Id] [bigint] NULL,
	[Elective_Batch_Id] [bigint] NULL,
	[Is_Section_Assigned] [bit] NULL,
	[Fees_Exemption] [nvarchar](50) NULL,
	[Mention_FeesExemption] [nvarchar](50) NULL,
	[AdmittedThrough] [nvarchar](100) NULL,
	[FatherName] [nvarchar](100) NULL,
	[Gender] [nvarchar](100) NULL,
	[dateofbirth] [datetime] NULL,
	[Religion] [nvarchar](100) NULL,
	[Community] [nvarchar](100) NULL,
	[Caste] [nvarchar](100) NULL,
	[MotherTongue] [nvarchar](100) NULL,
	[Bloodgroup] [nvarchar](100) NULL,
	[Height] [nvarchar](100) NULL,
	[Weight] [nvarchar](100) NULL,
	[Emailid] [nvarchar](100) NULL,
	[MobileNo] [nvarchar](100) NULL,
	[LandlineNo] [nvarchar](100) NULL,
	[PermenentAddress] [nvarchar](max) NULL,
	[City] [nvarchar](100) NULL,
	[Pincode] [nvarchar](100) NULL,
	[CountryId] [bigint] NULL,
	[StateId] [bigint] NULL,
	[PhoneNumber] [nvarchar](100) NULL,
	[LocalPresentAddress] [nvarchar](max) NULL,
	[C_City] [nvarchar](100) NULL,
	[C_Pincode] [nvarchar](100) NULL,
	[C_CountryId] [bigint] NULL,
	[C_StateId] [bigint] NULL,
	[C_PhoneNo] [nvarchar](100) NULL,
	[Passportno] [nvarchar](100) NULL,
	[BankAccountNo] [nvarchar](100) NULL,
	[BankName] [nvarchar](100) NULL,
	[AadhaarName] [nvarchar](100) NULL,
	[Examination_Prepare] [nvarchar](max) NULL,
	[InfluenceinEnglish] [nvarchar](100) NULL,
	[SelfImprove_Description] [nvarchar](100) NULL,
	[Languageknown] [nvarchar](100) NULL,
	[TC_ISSUED] [bit] NULL,
	[TC_REQUESTED_DATE] [datetime] NULL,
	[TC_ISSUED_DATE] [datetime] NULL,
	[Passportsizephoto] [bit] NULL,
	[Reported_Status] [nvarchar](15) NULL,
	[Reported_Date] [datetime] NULL,
	[Delete_Remarks] [nvarchar](500) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
	[AdmissionType] [nvarchar](50) NULL,
	[BusCode] [nvarchar](50) NULL,
	[BusBoard] [nvarchar](250) NULL,
	[AUAdmissionRegno] [nvarchar](50) NULL,
	[Consortium_AppNo] [nvarchar](50) NULL,
	[Consortium_RegNo] [nvarchar](50) NULL,
	[TNEA_ApplicationNo] [nvarchar](50) NULL,
	[TNEA_RecieptDate] [datetime] NULL,
	[AdmissionYear] [nvarchar](150) NULL,
	[Martial_Status] [nvarchar](50) NULL,
	[Nativity] [varchar](50) NULL,
	[Computer_Proficiency] [nvarchar](250) NULL,
	[IsFirstGraduate] [bit] NULL,
	[District] [nvarchar](50) NULL,
	[C_District] [nvarchar](50) NULL,
	[Nationality] [varchar](50) NULL,
	[MBA_ElectiveCourse] [varchar](50) NULL,
	[CoCurricular_Activities] [nvarchar](500) NULL,
	[ExtraCurricular_Activities] [nvarchar](500) NULL,
	[SiblingsStudied] [nvarchar](50) NULL,
	[Stu_Photo_Path] [nvarchar](1000) NULL,
	[Specialization] [bigint] NULL,
 CONSTRAINT [PK_tbl_cfg_StudentPersonalDetail] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_SUBMODULE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CFG_SUBMODULE](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[SubModuleIcon] [nvarchar](50) NULL,
	[ModuleId] [int] NOT NULL,
	[SubModuleName] [varchar](50) NULL,
	[SubModuleDesc] [varchar](500) NULL,
	[OrderBy] [int] NULL,
	[IsDeleted] [bit] NULL,
	[CreatedBy] [int] NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [int] NULL,
	[ModifiedDate] [datetime] NULL,
 CONSTRAINT [PK_SubModule] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_THEME_SETTING]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_THEME_SETTING](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[Wallpaper_Name] [nvarchar](50) NULL,
	[Wallpaper_Path] [nvarchar](100) NULL,
	[Menu_Type] [nvarchar](30) NULL,
	[Color_Type] [nvarchar](50) NULL,
	[Is_Active] [bit] NULL,
	[Created_By] [int] NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [int] NULL,
	[Modified_Date] [datetime] NULL,
	[Choose_Skin] [nvarchar](20) NULL,
	[Fixed_Navbar] [bit] NULL,
	[Fixed_Sidebar] [bit] NULL,
	[Fixed_Breadcrumbs] [bit] NULL,
	[Right_to_Left] [bit] NULL,
	[Inside_Container] [bit] NULL,
 CONSTRAINT [PK_theme_Setting] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_TIMETABLE_DEFINITION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_TIMETABLE_DEFINITION](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Group_Id] [bigint] NOT NULL,
	[Institution_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[Section_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NOT NULL,
	[AcademicYear_Id] [bigint] NOT NULL,
	[Regulation_Id] [bigint] NULL,
	[ClassInChargeStaff_Id] [bigint] NOT NULL,
	[Batch_Id] [bigint] NULL,
	[CLASS_CODE] [nvarchar](15) NULL,
	[Day] [nvarchar](15) NULL,
	[Period] [nvarchar](100) NULL,
	[From_Period] [nvarchar](10) NULL,
	[To_Period] [nvarchar](10) NULL,
	[Start_Time] [nvarchar](15) NULL,
	[End_Time] [nvarchar](15) NULL,
	[Priority] [nvarchar](15) NULL,
	[Course_Id] [bigint] NOT NULL,
	[TutorialType] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_TimeTable_Definition] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_TIMETABLE_MASTER]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_TIMETABLE_MASTER](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Group_Id] [bigint] NOT NULL,
	[Institution_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[DIVISION_CODE] [nvarchar](15) NULL,
	[CLASS_CODE] [nvarchar](15) NULL,
	[DAY] [nvarchar](10) NULL,
	[SAT_WORKING_PERIOD] [nvarchar](100) NULL,
	[SAT_WORKING_NO_OF_PERIOD] [int] NULL,
	[SAT_NO_OF_INTERVAL] [int] NULL,
	[PERIOD] [nvarchar](100) NULL,
	[NO_OF_PERIOD] [int] NULL,
	[NO_OF_INTERVAL] [int] NULL,
	[INTERVAL_AFTER_PERIOD] [nvarchar](50) NULL,
	[HALFDAY_HOURS] [nvarchar](50) NULL,
	[START_TIME] [nvarchar](10) NULL,
	[END_TIME] [nvarchar](10) NULL,
	[USER_ID] [nvarchar](15) NULL,
	[PERIOD_TYPE] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_cfg_TableMaster] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_TOPPERS_COUNT]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_TOPPERS_COUNT](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Count] [int] NULL,
 CONSTRAINT [PK_T_CFG_TOPPERS_COUNT] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_USER_ROLE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_USER_ROLE](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Role_Id] [bigint] NOT NULL,
	[User_Id] [bigint] NOT NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_User_Role] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_USER_TYPE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_USER_TYPE](
	[User_Type_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[User_Type_Name] [nvarchar](100) NULL,
	[User_Type_Status] [bit] NULL,
	[User_Type_Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_User] PRIMARY KEY CLUSTERED 
(
	[User_Type_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_USERPERMISSION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_USERPERMISSION](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[UserId] [int] NOT NULL,
	[PrivilegeId] [int] NOT NULL,
	[ModuleId] [int] NOT NULL,
	[SubModuleId] [int] NOT NULL,
	[ScreenId] [int] NOT NULL,
	[bCreate] [bit] NULL,
	[bView] [bit] NULL,
	[bUpdate] [bit] NULL,
	[bDelete] [bit] NULL,
	[IsDeleted] [bit] NULL,
	[CreatedBy] [int] NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [int] NULL,
	[ModifiedDate] [datetime] NULL,
	[OrderBy] [int] NULL,
 CONSTRAINT [PK_UsersPermission] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_WIDGET]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CFG_WIDGET](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[WidgetName] [varchar](50) NULL,
	[Description] [varchar](50) NULL,
	[Roles] [nvarchar](50) NULL,
	[Status] [bit] NOT NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_T_CFG_WIDGET] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CFG_YEARBASED_EXAM]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_YEARBASED_EXAM](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Programme_Id] [bigint] NULL,
	[Year_Id] [nvarchar](50) NULL,
	[Exam_Id] [nvarchar](50) NULL,
 CONSTRAINT [PK_T_CFG_YEARBASED_EXAM] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CFG_YEAROFSTUDY_MASTER]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CFG_YEAROFSTUDY_MASTER](
	[YearOfStudy_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[YearOfStudy_Code] [nvarchar](15) NOT NULL,
	[YearOfStudy_Name] [nvarchar](50) NULL,
	[Programme_Id] [bigint] NOT NULL,
	[CurrentRegulation] [bigint] NULL,
	[ActiveExam_Id] [bigint] NULL,
	[Active_Attendance_Exam_Id] [bigint] NULL,
	[AvailableExam_Ids] [nvarchar](15) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_Year] PRIMARY KEY CLUSTERED 
(
	[YearOfStudy_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CIRCULAR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CIRCULAR](
	[CIRCULAR_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[DEPARTMENT_ID] [bigint] NULL,
	[CIRCULAR_DATE] [datetime] NOT NULL,
	[CIRCULAR_TYPE] [nvarchar](50) NOT NULL,
	[CIRCULAR_DESCRIPTION] [nvarchar](max) NULL,
	[CIRCULAR_FILE_PATH] [nvarchar](max) NULL,
	[From_Date] [datetime] NULL,
	[To_Date] [datetime] NULL,
	[STATUS] [bit] NOT NULL,
	[FLAG] [bit] NOT NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_tbl_circular] PRIMARY KEY CLUSTERED 
(
	[CIRCULAR_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CONFERENCE_HALL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_CONFERENCE_HALL](
	[Hall_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[status] [bit] NULL,
	[Type] [bigint] NULL,
	[hallcode] [varchar](50) NULL,
	[hallno] [varchar](50) NULL,
	[description] [varchar](max) NULL,
	[Seat] [int] NULL,
	[location] [varchar](50) NULL,
	[Deleteflag] [bit] NULL,
	[Created_By] [varchar](50) NULL,
	[Created_date] [datetime] NULL,
	[Modified_By] [varchar](50) NULL,
	[Modified_date] [datetime] NULL,
 CONSTRAINT [PK_T_ConferenceHall] PRIMARY KEY CLUSTERED 
(
	[Hall_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_CONFERENCEEVENTS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CONFERENCEEVENTS](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[STUDENTTABLE_ID] [bigint] NOT NULL,
	[FROM_DATE_ACTIVITY] [datetime] NULL,
	[TO_DATE_ACTIVITY] [datetime] NULL,
	[TYPE_OF_Conference] [nvarchar](50) NULL,
	[TitleOfThePaper] [nvarchar](50) NULL,
	[NAME_OF_ORGANISTION] [nvarchar](250) NULL,
	[RECOGNISTION_STATUS] [nvarchar](50) NULL,
	[REMARKS] [nvarchar](max) NULL,
	[STATUS] [bit] NULL,
	[FLAG] [bit] NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_tbl_ConferencesAttended] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CONFERENCEHALLBOOKING_DTL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CONFERENCEHALLBOOKING_DTL](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Hdr_Id] [bigint] NOT NULL,
	[Hall_Id] [bigint] NOT NULL,
	[Approval_Status] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_by] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_conference_booking_Dtl] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_CONFERENCEHALLBOOKING_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_CONFERENCEHALLBOOKING_HDR](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[From_Date] [datetime] NULL,
	[To_Date] [datetime] NULL,
	[From_Time] [nvarchar](15) NULL,
	[To_Time] [nvarchar](15) NULL,
	[Description] [nvarchar](max) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_by] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_conference_Booking] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_COUNSELING]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_COUNSELING](
	[Counselingid] [bigint] IDENTITY(1,1) NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[academicyear_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[TestType] [bigint] NULL,
	[CounsellingType] [nvarchar](100) NULL,
	[DisciplineType] [nvarchar](500) NULL,
	[TestDate] [datetime] NULL,
	[Topic] [varchar](max) NULL,
	[Suggestion] [varchar](max) NULL,
	[Action] [varchar](max) NULL,
	[NoofArrears] [varchar](max) NULL,
	[PreviousSemesterArrears] [varchar](max) NULL,
	[OverallPercentage] [varchar](max) NULL,
	[Documents_Path] [nvarchar](250) NULL,
	[Counselling_Review_Date] [datetime] NULL,
	[Remarks] [nvarchar](max) NULL,
	[Mentor_Remarks] [nvarchar](max) NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_date] [datetime] NULL,
	[Status] [bit] NULL,
	[Deleteflag] [bit] NULL,
 CONSTRAINT [PK_tbl_Counseling] PRIMARY KEY CLUSTERED 
(
	[Counselingid] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_COURSE_REGULATION_SYALLBUS_DTL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_COURSE_REGULATION_SYALLBUS_DTL](
	[Identity] [bigint] IDENTITY(1,1) NOT NULL,
	[ID] [bigint] NOT NULL,
	[Course_Code] [bigint] NOT NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[FLAG] [bit] NOT NULL,
	[STATUS] [bit] NOT NULL,
 CONSTRAINT [PK_tbl_subject_Regulation_syallbus_Dtl] PRIMARY KEY CLUSTERED 
(
	[Identity] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_COURSE_REGULATION_SYALLBUS_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_COURSE_REGULATION_SYALLBUS_HDR](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[Regulation_ID] [bigint] NOT NULL,
	[Institution_ID] [bigint] NOT NULL,
	[Branch_ID] [bigint] NOT NULL,
	[YearOfStudy_ID] [bigint] NOT NULL,
	[Semester_ID] [bigint] NOT NULL,
	[status] [bit] NOT NULL,
	[Flag] [bit] NOT NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_subject_Regulation_syallbus] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_COURSEGROUPING]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_COURSEGROUPING](
	[CourseGrouping_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Regulation_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NOT NULL,
	[Course_Id] [bigint] NULL,
	[CourseGrouping_Code] [nvarchar](15) NULL,
	[Status] [nvarchar](8) NULL,
	[Delete_Flag] [int] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_cfg_SubjectGrouping] PRIMARY KEY CLUSTERED 
(
	[CourseGrouping_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_COURSESYALLABUSDETAILS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_COURSESYALLABUSDETAILS](
	[CourseSyllabusID] [bigint] IDENTITY(1,1) NOT NULL,
	[CourseId] [bigint] NOT NULL,
	[Unit] [nvarchar](50) NULL,
	[Unitdesc] [nvarchar](max) NULL,
	[period] [int] NULL,
	[IsDeleted] [bit] NULL,
	[flag] [bit] NULL,
 CONSTRAINT [PK_SubSyallabusDetails] PRIMARY KEY CLUSTERED 
(
	[CourseSyllabusID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_DAILYUPDATES]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_DAILYUPDATES](
	[id] [bigint] IDENTITY(1,1) NOT NULL,
	[InstitutionGroupid] [bigint] NULL,
	[InstitutionId] [bigint] NULL,
	[Events] [nvarchar](250) NULL,
	[Description] [nvarchar](250) NULL,
	[IsActive] [bit] NULL,
	[IsDeleted] [bit] NULL,
	[CreatedDate] [datetime] NULL,
	[Createdby] [bigint] NULL,
 CONSTRAINT [PK_tbl_DailyUpdates] PRIMARY KEY CLUSTERED 
(
	[id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_DISCILPLINARY_TYPE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_DISCILPLINARY_TYPE](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[DISCIPLINARY_TYPE_CODE] [nvarchar](50) NULL,
	[DISCIPLINARY_TYPE__NAME] [nvarchar](50) NULL,
	[FLAG] [bit] NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_t_Displinary_Type] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_DISCIPLINARY_ACTIONS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_DISCIPLINARY_ACTIONS](
	[DISCIPLINARY_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[ACADEMIC_YEAR_ID] [bigint] NOT NULL,
	[Branch_ID] [bigint] NULL,
	[YEAROFSTUDY_ID] [bigint] NULL,
	[SEMESTER_ID] [bigint] NULL,
	[SECTION_ID] [bigint] NULL,
	[DISCIPLINARY_TYPE] [bigint] NOT NULL,
	[DESCRIPTION] [nvarchar](max) NOT NULL,
	[REMARKS] [nvarchar](max) NULL,
	[NO_OF_DAYS] [nvarchar](50) NULL,
	[FLAG] [bit] NOT NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_tbl_Disciplinary_Action] PRIMARY KEY CLUSTERED 
(
	[DISCIPLINARY_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_DISCIPLINARY_ACTIONS_DTL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_DISCIPLINARY_ACTIONS_DTL](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[DISCIPLINARY_ID] [bigint] NOT NULL,
	[STUDENTTABLE_ID] [bigint] NOT NULL,
	[DOCUMENT_PATH] [nvarchar](max) NULL,
	[DELETE_FLAG] [bit] NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_tbl_disciplinary_action_Dtl] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_EDUMATE_REPORTS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_EDUMATE_REPORTS](
	[ReportId] [int] IDENTITY(1,1) NOT NULL,
	[ReportCode] [varchar](100) NULL,
	[ReportName] [nvarchar](500) NULL,
	[Description] [nvarchar](500) NULL,
	[Active] [bit] NULL,
	[DeleteFlag] [bit] NULL,
	[DataDate] [datetime] NULL,
 CONSTRAINT [PK_EdumateReports] PRIMARY KEY CLUSTERED 
(
	[ReportId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_ELECTIVEBATCHTEXT]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_ELECTIVEBATCHTEXT](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Programme_Id] [bigint] NULL,
	[Year_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[Elective_Text] [nvarchar](200) NULL,
 CONSTRAINT [PK_T_MBA] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_EXAM_DEFINITION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_EXAM_DEFINITION](
	[Exam_Definition_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Exam_Id] [bigint] NOT NULL,
	[Academic_Year_Id] [bigint] NOT NULL,
	[Regulation_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NOT NULL,
	[Section_Id] [nvarchar](50) NULL,
	[Counselling_CutOff_Date] [datetime] NULL,
	[Attendance_FromDate] [datetime] NULL,
	[Attendance_ToDate] [datetime] NULL,
	[ClassAfterTest] [bit] NULL,
	[AddLab] [bit] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Exam_Definition] PRIMARY KEY CLUSTERED 
(
	[Exam_Definition_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_EXAMDEFINITION_DETAIL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_EXAMDEFINITION_DETAIL](
	[Exam_Dtl_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Exam_Definition_Id] [bigint] NOT NULL,
	[Course_Id] [bigint] NOT NULL,
	[Exam_Date] [datetime] NULL,
	[Mark_Submission_Date] [datetime] NULL,
	[ReTest_Mark_Submission_Date] [datetime] NULL,
	[Exam_Periods] [nvarchar](50) NULL,
	[status] [bit] NULL,
	[Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_ExamDefinition_Detail] PRIMARY KEY CLUSTERED 
(
	[Exam_Dtl_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_EXTRA_CIRCULAR_ACTIVITY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_EXTRA_CIRCULAR_ACTIVITY](
	[EXTRA_CIRUCULAR_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[STUDENTTABLE_ID] [bigint] NOT NULL,
	[FROM_DATE_ACTIVITY] [datetime] NULL,
	[TO_DATE_ACTIVITY] [datetime] NULL,
	[TYPE_OF_EVENTS] [nvarchar](50) NULL,
	[EVENT_NAME] [nvarchar](50) NULL,
	[NAME_OF_ORGANISTION] [nvarchar](250) NULL,
	[RECOGNISTION_STATUS] [nvarchar](50) NULL,
	[REMARKS] [nvarchar](max) NULL,
	[STATUS] [bit] NULL,
	[FLAG] [bit] NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_tbl_Extra_circular_Activity] PRIMARY KEY CLUSTERED 
(
	[EXTRA_CIRUCULAR_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_FEES_EXCEMPTION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_FEES_EXCEMPTION](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Academic_Year_Id] [bigint] NULL,
	[StudentTable_Id] [bigint] NULL,
	[FeesType_Id] [bigint] NULL,
	[Student_Name] [nvarchar](100) NULL,
	[Branch_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[CGPA_RangeId] [int] NULL,
	[Amount] [decimal](18, 2) NULL,
	[Remarks] [nvarchar](500) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Fees_Excemption] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_HOLIDAY_DTL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_HOLIDAY_DTL](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Hdr_Id] [bigint] NULL,
	[Branch_Id] [bigint] NULL,
	[Year_Id] [bigint] NULL,
	[Active] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
 CONSTRAINT [PK_T_HOLIDAY_DTL] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_HOLIDAY_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_HOLIDAY_HDR](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Programme_Id] [bigint] NULL,
	[From_date] [datetime] NULL,
	[To_date] [datetime] NULL,
	[Dayname] [varchar](50) NULL,
	[Description] [nvarchar](500) NULL,
	[Active] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
 CONSTRAINT [PK_T_HOLIDAY_HDR] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_HOLIDAYS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_HOLIDAYS](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[PROGRAMME_ID] [bigint] NULL,
	[BRANCH_ID] [bigint] NULL,
	[YEAR_ID] [bigint] NULL,
	[LEAVE_DATE] [datetime] NULL,
	[DAY_NAME] [nvarchar](20) NULL,
	[DESCRIPTION] [nvarchar](500) NULL,
	[STATUS] [bit] NULL,
	[DELETE_FLAG] [bit] NULL,
	[CREATED_DATE] [datetime] NULL,
	[CREATED_BY] [nvarchar](100) NULL,
	[MODIFIED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](100) NULL,
 CONSTRAINT [PK_T_HOLIDAYS] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_INPLANT_DETAILS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_INPLANT_DETAILS](
	[id] [bigint] IDENTITY(1,1) NOT NULL,
	[InplantId] [bigint] NULL,
	[studentTableId] [bigint] NULL,
 CONSTRAINT [PK_tbl_Inplant_Details] PRIMARY KEY CLUSTERED 
(
	[id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_INPLANT_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_INPLANT_HDR](
	[InplantId] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Id] [bigint] NULL,
	[BranchId] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[SemesterId] [bigint] NOT NULL,
	[Section_Id] [bigint] NOT NULL,
	[Organisationid] [bigint] NOT NULL,
	[FromDate] [datetime] NULL,
	[ToDate] [datetime] NULL,
	[Description] [nvarchar](max) NULL,
	[Status] [bit] NULL,
	[IsDeleted] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Inplant_Hdr] PRIMARY KEY CLUSTERED 
(
	[InplantId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_INPLANTTRAINING]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_INPLANTTRAINING](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[InstitutionID] [bigint] NULL,
	[BranchID] [bigint] NULL,
	[YearID] [bigint] NULL,
	[SemesterID] [bigint] NULL,
	[SectionID] [bigint] NULL,
	[StudentID] [bigint] NULL,
	[OrganisationName] [nvarchar](150) NULL,
	[OrganisationAddress] [nvarchar](max) NULL,
	[FromDate] [datetime] NULL,
	[ToDate] [datetime] NULL,
	[Description] [nvarchar](max) NULL,
	[Documents_Path] [nvarchar](500) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[CreatedBy] [nvarchar](100) NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [nvarchar](100) NULL,
	[ModifiedDate] [datetime] NULL,
 CONSTRAINT [PK__tbl_Inpl__3214EC277F490C64] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_INTERVIEW_PROCESS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_INTERVIEW_PROCESS](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Process_Code] [nvarchar](50) NULL,
	[Description] [nvarchar](150) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_T_INTERVIEW_PROCESS] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_INTERVIEW_STATUS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_INTERVIEW_STATUS](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Code] [nvarchar](50) NULL,
	[Description] [nvarchar](150) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_T_INTERVIEW_STATUS] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_INTERVIEW_STUDENTS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_INTERVIEW_STUDENTS](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Recruiter_Id] [bigint] NULL,
	[Student_Id] [bigint] NULL,
	[Interview_Mode] [varchar](50) NULL,
	[IsAppeared] [bit] NULL,
	[InterviewStatus_Id] [bigint] NULL,
	[Active] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Data_Date] [datetime] NULL,
 CONSTRAINT [PK_T_INTERVIEW_STUDENTS] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_LEAVEDETAILS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_LEAVEDETAILS](
	[LeaveId] [bigint] IDENTITY(1,1) NOT NULL,
	[Student_Id] [bigint] NOT NULL,
	[From_Date] [datetime] NULL,
	[To_Date] [datetime] NULL,
	[Leave_Periods] [nvarchar](50) NULL,
	[Leave_Time] [nvarchar](100) NULL,
	[Leave_Remarks] [nvarchar](max) NULL,
	[Leave_Approve] [bit] NULL,
	[LeaveReasonId] [bigint] NOT NULL,
	[Leave_ApprovedRemark] [nvarchar](max) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Leave_Details] PRIMARY KEY CLUSTERED 
(
	[LeaveId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_MARK_ENTRY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_MARK_ENTRY](
	[Mark_Entry_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Id] [bigint] NULL,
	[AcademicYear_Id] [bigint] NULL,
	[StaffClassCourseDtl_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NOT NULL,
	[Course_Id] [bigint] NOT NULL,
	[Exam_Id] [bigint] NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[Is_OD] [bit] NULL,
	[Is_Absent] [bit] NULL,
	[Is_Absent_Retest] [bit] NULL,
	[Marks] [nvarchar](50) NULL,
	[RetestMarks] [nvarchar](50) NULL,
	[PreviousMarks] [nvarchar](50) NULL,
	[TestType] [nvarchar](50) NULL,
	[Remarks] [nvarchar](max) NULL,
	[Retest_Remarks] [nvarchar](max) NULL,
	[ElectiveBatch_Id] [bigint] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Marks_Entry] PRIMARY KEY CLUSTERED 
(
	[Mark_Entry_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_MARKDECLARATION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_MARKDECLARATION](
	[MarkId] [int] IDENTITY(1,1) NOT NULL,
	[Course] [nvarchar](50) NULL,
	[Sem1] [nvarchar](50) NULL,
	[Sem2] [nvarchar](50) NULL,
	[Sem3] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_MarkDeclaration] PRIMARY KEY CLUSTERED 
(
	[MarkId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_MBA_SPECIALIZATION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_MBA_SPECIALIZATION](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Code] [nvarchar](50) NULL,
	[Specialization] [nvarchar](100) NULL,
	[Active] [bit] NULL,
	[Delete_Flag] [bit] NULL,
 CONSTRAINT [PK_T_MBA_SPECIALIZATION] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_MENTOR_DTL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_MENTOR_DTL](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[MENTOR_ID] [bigint] NOT NULL,
	[STUDENTTABLE_ID] [bigint] NOT NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_t_Mentor_dtl] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_MENTOR_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_MENTOR_HDR](
	[MENTOR_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[INSTITUTION_ID] [bigint] NULL,
	[Department_Id] [bigint] NULL,
	[BRANCH_ID] [bigint] NULL,
	[YEAR_ID] [bigint] NULL,
	[SEMESTER_ID] [bigint] NULL,
	[SECTION_ID] [bigint] NULL,
	[STAFF_ID] [bigint] NOT NULL,
	[StudentBatch_Id] [bigint] NULL,
	[STATUS] [bit] NOT NULL,
	[FLAG] [bit] NOT NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_t_Mentor_Hdr] PRIMARY KEY CLUSTERED 
(
	[MENTOR_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_MENTOR_HISTORY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_MENTOR_HISTORY](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[StudentTable_Id] [bigint] NULL,
	[Branch_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[Staff_Id] [bigint] NULL,
	[From_Date] [datetime] NULL,
	[To_Date] [datetime] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[CreatedBy] [nvarchar](50) NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [nvarchar](50) NULL,
	[ModifiedDate] [datetime] NULL,
 CONSTRAINT [PK_T_MENTOR_HISTORY] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_MERITSCHOLARSHIP]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_MERITSCHOLARSHIP](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Academic_Year_Id] [bigint] NULL,
	[StudentTable_Id] [bigint] NULL,
	[Student_Name] [nvarchar](100) NULL,
	[Branch_Id] [bigint] NULL,
	[Year_Of_Study_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[CGPA_RangeId] [int] NULL,
	[Amount] [decimal](18, 0) NULL,
	[Remarks] [nvarchar](500) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_MeritScholarship] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_MESS_REPORT]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_MESS_REPORT](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[StudentId_Rollno] [nvarchar](100) NULL,
	[Lunch_Time] [datetime] NULL,
	[Lunch_Date] [datetime] NULL,
 CONSTRAINT [PK_T_MESS_REPORT] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_MESS_STUDENT_DETAILS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_MESS_STUDENT_DETAILS](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[AcademicYear_Id] [bigint] NULL,
	[StudentTable_Id] [bigint] NULL,
	[StudentId_Rollno] [nvarchar](100) NULL,
	[UniversityRegNo] [nvarchar](100) NULL,
	[StudentName] [nvarchar](100) NULL,
	[Branch_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[Is_Hostel] [bit] NULL,
	[Eligibility] [nvarchar](5) NULL,
	[Payment_Done] [nvarchar](5) NULL,
	[Lunch_Time] [datetime] NULL,
	[HadLunch] [bit] NULL,
 CONSTRAINT [PK_T_MESS_STUDENT_DETAILS] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_NEWCOMPANY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_NEWCOMPANY](
	[ID] [int] IDENTITY(1,1) NOT NULL,
	[CompanyName] [nvarchar](50) NULL,
	[CompanySite] [nvarchar](50) NULL,
	[LogoPath] [nvarchar](max) NULL,
	[CompanyPresentation] [nvarchar](max) NULL,
	[Description] [nvarchar](max) NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_T_NEWCOMPANY] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_NO_DUES_DEPARTMENT]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_NO_DUES_DEPARTMENT](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Related_To_Department] [nvarchar](100) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_No_Dues_Department] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_NO_DUES_DETAILS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_NO_DUES_DETAILS](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Student_Table_Id] [bigint] NOT NULL,
	[No_Dues_Table_Id] [bigint] NOT NULL,
	[Is_Cleared] [bit] NULL,
	[Remarks] [nvarchar](max) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_No_Dues_Details] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_OD_DETAILS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_OD_DETAILS](
	[ODId] [bigint] IDENTITY(1,1) NOT NULL,
	[Student_Id] [bigint] NOT NULL,
	[From_Date] [datetime] NULL,
	[To_Date] [datetime] NULL,
	[OD_Periods] [nvarchar](50) NULL,
	[OD_Remarks] [nvarchar](max) NULL,
	[OD_Approve] [bit] NULL,
	[OD_ApproveRemark] [nvarchar](max) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Od_Details] PRIMARY KEY CLUSTERED 
(
	[ODId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_OD_DETAILS_DTL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_OD_DETAILS_DTL](
	[ODDtl_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[ODHdr_Id] [bigint] NULL,
	[StudentTable_Id] [bigint] NULL,
	[OD_Approve] [bit] NULL,
	[Delete_Flag] [bit] NULL,
 CONSTRAINT [PK_T_OD_DETAILS_DTL] PRIMARY KEY CLUSTERED 
(
	[ODDtl_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_OD_DETAILS_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_OD_DETAILS_HDR](
	[ODId] [bigint] IDENTITY(1,1) NOT NULL,
	[From_Date] [datetime] NULL,
	[To_Date] [datetime] NULL,
	[OD_Periods] [nvarchar](50) NULL,
	[OD_Remarks] [nvarchar](max) NULL,
	[OD_ApproveRemark] [nvarchar](max) NULL,
	[Common_Od] [bit] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Od_Details_HDR] PRIMARY KEY CLUSTERED 
(
	[ODId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_PARENT_REGISTRATION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_PARENT_REGISTRATION](
	[Parent_ID] [bigint] IDENTITY(7001,1) NOT NULL,
	[REGISTERATION_TYPE] [nvarchar](50) NOT NULL,
	[RELATIONSHIP] [nvarchar](50) NOT NULL,
	[TITLE] [nvarchar](50) NOT NULL,
	[FIRST_NAME] [nvarchar](50) NOT NULL,
	[LAST_NAME] [nvarchar](50) NOT NULL,
	[GENDER] [nvarchar](50) NOT NULL,
	[DOB] [datetime] NOT NULL,
	[ADDRESS] [nvarchar](250) NOT NULL,
	[CITY] [nvarchar](50) NOT NULL,
	[PINCODE] [nvarchar](8) NOT NULL,
	[STATE] [nvarchar](50) NOT NULL,
	[COUNTRY] [nvarchar](50) NOT NULL,
	[HOME_PHONE] [nvarchar](15) NULL,
	[CELL_PHONE] [nvarchar](15) NOT NULL,
	[ALTERNATE_NUMBE] [nvarchar](15) NULL,
	[EMAIL_ID] [nvarchar](50) NOT NULL,
	[PASSWORD] [nvarchar](50) NOT NULL,
	[CONFIRM_PASSWORD] [nvarchar](50) NULL,
	[JOB_POSITION] [nvarchar](250) NULL,
	[JOB_COMPANY] [nvarchar](250) NULL,
	[JOB_INDUSTRY_TYPE] [nvarchar](250) NULL,
	[Employement_Type] [nvarchar](50) NULL,
	[Name_Of_Business] [nvarchar](500) NULL,
	[Highest_Education] [nvarchar](50) NULL,
	[EDU_COURSE] [nvarchar](250) NULL,
	[EDU_INSTITUTION] [nvarchar](250) NULL,
	[EDU_MAJOR_SUBJECT] [nvarchar](250) NULL,
	[EDU_YEAR_OF_PASSING] [nvarchar](50) NULL,
	[STATUS] [bit] NOT NULL,
	[FLAG] [bit] NOT NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
	[Parent_Image] [image] NULL,
 CONSTRAINT [PK_tbl_Parent_registeration] PRIMARY KEY CLUSTERED 
(
	[Parent_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_PARENT_TO_STUDENT]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_PARENT_TO_STUDENT](
	[ID] [int] IDENTITY(1,1) NOT NULL,
	[PARENT_ID] [bigint] NOT NULL,
	[StudentId] [bigint] NOT NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_tbl_Parent_To_Student] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_PG_DEGREE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_PG_DEGREE](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[PG_Degree_Name] [nvarchar](10) NULL,
 CONSTRAINT [PK_T_PG_DEGREE] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_PLACEMENTS_DETAILS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_PLACEMENTS_DETAILS](
	[id] [bigint] IDENTITY(1,1) NOT NULL,
	[PlacementId] [bigint] NOT NULL,
	[BranchId] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[SemesterId] [bigint] NOT NULL,
	[Section_Id] [bigint] NOT NULL,
	[studentTableId] [bigint] NOT NULL,
	[InterviewStatus] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[IsDeleted] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_Placement_Details] PRIMARY KEY CLUSTERED 
(
	[id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_PLACEMENTS_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_PLACEMENTS_HDR](
	[id] [bigint] IDENTITY(1,1) NOT NULL,
	[Organisationid] [bigint] NOT NULL,
	[InterviewType] [nvarchar](50) NULL,
	[Trained_By_Id] [bigint] NULL,
	[InterviewDate] [datetime] NULL,
	[AnnualPakage] [decimal](18, 0) NULL,
	[Remarks] [text] NULL,
	[Status] [bit] NULL,
	[IsDeleted] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_Placement_Hdr] PRIMARY KEY CLUSTERED 
(
	[id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_PROJECT_DTL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_PROJECT_DTL](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[PROJECT_ID] [bigint] NOT NULL,
	[STUDENTTABLE_ID] [bigint] NOT NULL,
	[STUDENT_NAME] [nvarchar](50) NOT NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_t_Project_Dtl] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_PROJECT_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_PROJECT_HDR](
	[PROJECT_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[ACADEMIC_YEAR_ID] [bigint] NOT NULL,
	[PROJECT_TITLE] [nvarchar](200) NOT NULL,
	[PROJECT_DESCRIPTION] [nvarchar](max) NOT NULL,
	[PROJECT_GUIDE_NAME] [nvarchar](100) NOT NULL,
	[CRAETED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
	[STATUS] [bit] NOT NULL,
	[FLAG] [nvarchar](50) NULL,
	[InterviewDate] [datetime] NULL,
 CONSTRAINT [PK_t_Project_Hdr] PRIMARY KEY CLUSTERED 
(
	[PROJECT_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_QUESTIONPAPERREVIEW]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_QUESTIONPAPERREVIEW](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Academicyear] [varchar](50) NULL,
	[branch_Id] [bigint] NULL,
	[semester_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[staffid] [varchar](50) NULL,
	[Course_id] [bigint] NULL,
	[questionpapercode] [varchar](max) NULL,
	[difficultylevel] [varchar](50) NULL,
	[reason] [varchar](max) NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Status] [bit] NULL,
	[DeleteFlag] [bit] NULL,
 CONSTRAINT [PK_tbl_QuestionPaperReview] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_RAISEQUERY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_RAISEQUERY](
	[RAISE_QUERY_ID] [bigint] IDENTITY(1,1) NOT NULL,
	[INSTITUTION_ID] [bigint] NULL,
	[QUERY_TYPE] [nvarchar](50) NULL,
	[Query_Topic] [nvarchar](max) NULL,
	[RAISED_BY] [bigint] NOT NULL,
	[STUDENTTABLE_ID] [bigint] NULL,
	[COURSE_ID] [bigint] NULL,
	[DESCRIPTION] [nvarchar](max) NOT NULL,
	[STATUS_QUERY] [nvarchar](50) NULL,
	[FLAG] [bit] NULL,
	[STATUS] [bit] NULL,
	[CREATED_BY] [nvarchar](50) NULL,
	[CREATED_DATE] [datetime] NULL,
	[MODIFIED_BY] [nvarchar](50) NULL,
	[MODIFIED_DATE] [datetime] NULL,
 CONSTRAINT [PK_tbl_Raise_Query] PRIMARY KEY CLUSTERED 
(
	[RAISE_QUERY_ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_RECRUITER]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_RECRUITER](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[AcademicYear_Id] [bigint] NULL,
	[Company_Id] [bigint] NULL,
	[Interview_Name] [nvarchar](500) NULL,
	[From_Date] [datetime] NULL,
	[To_Date] [datetime] NULL,
	[Eligible_Dept] [varchar](75) NULL,
	[Interview_Process_Id] [varchar](75) NULL,
	[NoofStudEligile] [varchar](50) NULL,
	[InterviewType] [varchar](50) NULL,
	[Package] [decimal](18, 0) NULL,
	[IsComplete] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
 CONSTRAINT [PK_T_RECRUITER] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_RETEST_ATTENDANCE_ENTRY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_RETEST_ATTENDANCE_ENTRY](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NOT NULL,
	[Section_Id] [bigint] NOT NULL,
	[Staff_Id] [bigint] NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[Course_Id] [bigint] NOT NULL,
	[StaffClassCourseDtl_Id] [bigint] NOT NULL,
	[Attendance_Date] [datetime] NULL,
	[Attendance_Day] [nvarchar](50) NULL,
	[Attendance_Type] [nvarchar](50) NULL,
	[Period] [nvarchar](50) NULL,
	[PeriodType] [nvarchar](100) NULL,
	[Exam_Id] [bigint] NULL,
	[Attendance_Status] [nvarchar](50) NULL,
	[Batch_Id] [bigint] NULL,
	[Comments] [nvarchar](500) NULL,
	[TopicsCovered] [nvarchar](500) NULL,
	[AttendanceEdit_Remark] [nvarchar](500) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_T_RETEST_ATTENDANCE_ENTRY] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_REVALUATION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_REVALUATION](
	[Revulation_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Registration_No] [nvarchar](50) NULL,
	[Result_Type_Id] [bigint] NULL,
	[Result_Published_Year_Id] [bigint] NULL,
	[Revulation_Fees_Amount] [decimal](18, 2) NULL,
	[Revulation_Fees_Receipt_No] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_Revaluation] PRIMARY KEY CLUSTERED 
(
	[Revulation_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_REVALUATIONCOURSES]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_REVALUATIONCOURSES](
	[Revaluation_CourseId] [bigint] IDENTITY(1,1) NOT NULL,
	[Revaluation_Id] [bigint] NULL,
	[AU_Result_Id] [bigint] NULL,
	[Course_Code] [nvarchar](50) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_Date] [datetime] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
 CONSTRAINT [PK_RevaluationCourses] PRIMARY KEY CLUSTERED 
(
	[Revaluation_CourseId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_ROLL_OVER_HISTORY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_ROLL_OVER_HISTORY](
	[ROLLOVERID] [bigint] IDENTITY(1,1) NOT NULL,
	[STUDENT_ID] [bigint] NULL,
	[BRANCH_ID] [bigint] NULL,
	[PREV_YEAR_ID] [bigint] NULL,
	[PREV_SEM_ID] [bigint] NULL,
	[PREV_SEC_ID] [bigint] NULL,
	[PREV_ACADEMIC_ID] [bigint] NULL,
	[ROLL_YEAR_ID] [bigint] NULL,
	[ROLL_SEM_ID] [bigint] NULL,
	[ROLL_SEC_ID] [bigint] NULL,
	[ROLL_ACADEMIC_ID] [bigint] NULL,
	[REMARKS] [nvarchar](max) NULL,
 CONSTRAINT [PK_T_ROLL_OVER_HISTORY] PRIMARY KEY CLUSTERED 
(
	[ROLLOVERID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_SECTION_HISTORY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_SECTION_HISTORY](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[STUDENT_ID] [bigint] NULL,
	[BRANCH_ID] [bigint] NULL,
	[YEAR_ID] [bigint] NULL,
	[SEMESTER_ID] [bigint] NULL,
	[SECTION_ID] [bigint] NULL,
	[ACADEMIC_ID] [bigint] NULL,
	[REMARKS] [nvarchar](max) NULL,
	[STATUS_FLAG] [bit] NULL,
	[DELETE_FLAG] [bit] NULL,
	[DATA_DATE] [datetime] NULL,
	[CREATED_BY] [nvarchar](50) NULL,
 CONSTRAINT [PK_T_SECTION_HISTORY] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_SEMESTER_PUBLISH_YEAR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_SEMESTER_PUBLISH_YEAR](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Academic_Batch_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Result_Pub_Yr_Id] [bigint] NULL,
 CONSTRAINT [PK_T_SEMESTER_PUBLISH_YEAR] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_SMS_CLIENTS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_SMS_CLIENTS](
	[Client_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Client_Code] [nvarchar](15) NOT NULL,
	[Client_Name] [nvarchar](100) NOT NULL,
	[Client_Address] [nvarchar](500) NULL,
	[Client_Logo_Path] [nvarchar](max) NULL,
	[Client_Email_ID] [nvarchar](100) NULL,
	[Client_Website] [nvarchar](max) NULL,
	[Client_Contact_No] [nvarchar](100) NULL,
	[API_URL] [nvarchar](max) NULL,
	[SMS_API_URL] [nvarchar](max) NULL,
	[REP_API_URL] [nvarchar](max) NULL,
	[Msg_Working_Key] [nvarchar](30) NULL,
	[Msg_Sender_Code] [nvarchar](10) NULL,
	[Msg_Sender_Password] [nvarchar](15) NULL,
	[Created_By] [nvarchar](15) NULL,
	[Created_Date] [datetime] NULL,
	[IsActive] [bit] NOT NULL,
	[Flag] [bit] NOT NULL,
 CONSTRAINT [PK_tbl_SMS_Clients] PRIMARY KEY CLUSTERED 
(
	[Client_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_SMS_REPORT_UPLOAD]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_SMS_REPORT_UPLOAD](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[User_Id] [varchar](50) NULL,
	[Client_Id] [varchar](50) NULL,
	[Msg_Id] [varchar](max) NULL,
	[CampaignName] [varchar](max) NULL,
	[Mobile_Number] [nvarchar](100) NULL,
	[SenderID] [varchar](50) NULL,
	[SentTime] [varchar](50) NULL,
	[LastUpdated] [varchar](50) NULL,
	[Acknowledgment] [varchar](50) NULL,
	[MessageText] [varchar](max) NULL,
	[SMSLength] [varchar](max) NULL,
	[CreditsCharged] [varchar](50) NULL,
	[Reference_Number] [varchar](50) NULL,
	[Provider] [varchar](50) NULL,
	[Location] [varchar](50) NULL,
	[Data_Date] [datetime] NULL,
	[Create_User] [varchar](50) NULL,
	[IsActive] [bit] NULL,
	[Flag] [bit] NULL,
 CONSTRAINT [PK_tbl_SMSReportupload] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_SMS_RESPONSE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_SMS_RESPONSE](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[User_Id] [bigint] NOT NULL,
	[Client_Id] [bigint] NOT NULL,
	[Message_GID] [nvarchar](50) NULL,
	[Msg_ID] [nvarchar](50) NULL,
	[API_Response] [nvarchar](100) NULL,
	[CampaignName] [varchar](max) NULL,
	[Mobile_Number] [nvarchar](100) NULL,
	[SenderID] [varchar](50) NULL,
	[SentDate] [datetime] NOT NULL,
	[SentTime] [nvarchar](50) NULL,
	[LastUpdated] [datetime] NULL,
	[Acknowledgment] [varchar](50) NULL,
	[MessageText] [varchar](max) NULL,
	[SMSLength] [varchar](max) NULL,
	[CreditsCharged] [varchar](50) NULL,
	[Reference_Number] [varchar](50) NULL,
	[Provider] [varchar](50) NULL,
	[Location] [varchar](50) NULL,
	[Data_Date] [datetime] NULL,
	[Create_User] [varchar](50) NULL,
	[IsActive] [bit] NOT NULL,
	[Flag] [bit] NOT NULL,
 CONSTRAINT [PK_tbl_SMS_Response] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_SMS_STATUS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_SMS_STATUS](
	[StatusId] [int] IDENTITY(1,1) NOT NULL,
	[Status] [nvarchar](100) NULL,
	[Order] [int] NULL,
	[IsActive] [bit] NULL,
	[Flag] [bit] NULL,
 CONSTRAINT [PK_tbl_Status] PRIMARY KEY CLUSTERED 
(
	[StatusId] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_SMS_TEMPLATES]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_SMS_TEMPLATES](
	[SNo] [int] IDENTITY(1,1) NOT NULL,
	[Template_Type] [varchar](50) NOT NULL,
	[Template_Details] [varchar](200) NOT NULL,
	[No_of_Parameters] [int] NOT NULL,
	[Parameter] [varchar](50) NULL,
	[DateParameters] [varchar](50) NULL,
	[Client_Id] [bigint] NOT NULL,
	[Template_Path] [nvarchar](500) NULL,
	[IsActive] [bit] NOT NULL,
	[Flag] [bit] NOT NULL,
 CONSTRAINT [PK_tbl_Templates] PRIMARY KEY CLUSTERED 
(
	[SNo] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_SPONSARSHIP_DTL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_SPONSARSHIP_DTL](
	[Dtl_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Hdr_Id] [bigint] NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[Document_Path] [nvarchar](max) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Sponsorship_Dtl] PRIMARY KEY CLUSTERED 
(
	[Dtl_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_SPONSARSHIP_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_SPONSARSHIP_HDR](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Branch_Id] [bigint] NULL,
	[YearOfStudy_Id] [bigint] NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NULL,
	[ConferenceType_Id] [bigint] NULL,
	[From_Date] [datetime] NULL,
	[To_Date] [datetime] NULL,
	[Conference_Name] [nvarchar](150) NULL,
	[Project_Details] [nvarchar](max) NULL,
	[Sponsorship_Type] [nvarchar](50) NULL,
	[Sponsored_By] [nvarchar](50) NULL,
	[Amount_Sponsored] [decimal](18, 0) NULL,
	[Remarks] [nvarchar](max) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Sponcership] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_DTL]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_DTL](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Hdr_Id] [bigint] NOT NULL,
	[Course_Id] [bigint] NOT NULL,
	[Class_Section_Id] [bigint] NOT NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_StaffAssigning_Class_Subject_Dtl] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_HDR]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_HDR](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[AcademicYear_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NOT NULL,
	[Staff_Id] [bigint] NOT NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_StaffAssigningSection] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_STAFF_CATEGORY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_STAFF_CATEGORY](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Staff_Catogary_Name] [nvarchar](500) NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
 CONSTRAINT [PK_tbl_Staff_Catogary] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_STAFF_EXPERIENCE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_STAFF_EXPERIENCE](
	[Staff_Experience_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Staff_Id] [bigint] NOT NULL,
	[Institution_Name] [nvarchar](500) NULL,
	[Branch_Name] [nvarchar](500) NULL,
	[Designation_Name] [nvarchar](100) NULL,
	[From_Month] [datetime] NULL,
	[To_Month] [datetime] NULL,
	[Course_Handled] [nvarchar](300) NULL,
	[Overall_Result] [decimal](18, 2) NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Staff_Experience] PRIMARY KEY CLUSTERED 
(
	[Staff_Experience_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_STAFF_QUALIFICATION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_STAFF_QUALIFICATION](
	[Staff_Qualification_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Staff_Id] [bigint] NOT NULL,
	[Institution_Name] [nvarchar](500) NULL,
	[Branch_Name] [nvarchar](500) NULL,
	[Achievements_If_Any] [nvarchar](max) NULL,
	[From_Month] [datetime] NULL,
	[To_Month] [datetime] NULL,
	[Marks_Or_Grade] [nvarchar](10) NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_Staff_Qualification] PRIMARY KEY CLUSTERED 
(
	[Staff_Qualification_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_STUDENTCERTIFICATE_CLEARENCE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_STUDENTCERTIFICATE_CLEARENCE](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[InstitutionID] [bigint] NULL,
	[BranchID] [bigint] NULL,
	[YearID] [bigint] NULL,
	[SemesterID] [bigint] NULL,
	[SectionID] [bigint] NULL,
	[StudentID] [bigint] NULL,
	[CertificateID] [bigint] NULL,
	[IssuedFlag] [bit] NULL,
	[DateOfIssuance] [datetime] NULL,
	[DeleteFlag] [bit] NULL,
	[CreatedBy] [nvarchar](100) NULL,
	[CreatedDate] [datetime] NULL,
	[ModifiedBy] [nvarchar](100) NULL,
	[ModifiedDate] [datetime] NULL,
 CONSTRAINT [PK_tbl_StudentCertificateClearence] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_TEMP_MARKENTRY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_TEMP_MARKENTRY](
	[Mark_Entry_Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Id] [bigint] NULL,
	[AcademicYear_Id] [bigint] NULL,
	[StaffClassCourseDtl_Id] [bigint] NOT NULL,
	[Branch_Id] [bigint] NOT NULL,
	[YearOfStudy_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NULL,
	[Section_Id] [bigint] NOT NULL,
	[Course_Id] [bigint] NOT NULL,
	[Exam_Id] [bigint] NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[Is_Absent] [bit] NULL,
	[Marks] [nvarchar](50) NULL,
	[RetestMarks] [nvarchar](50) NULL,
	[PreviousMarks] [nvarchar](50) NULL,
	[TestType] [nvarchar](50) NULL,
	[Remarks] [nvarchar](max) NULL,
	[ElectiveBatch_Id] [bigint] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_temp_marksEntry] PRIMARY KEY CLUSTERED 
(
	[Mark_Entry_Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_TEMPLATE_DOWNLOAD]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_TEMPLATE_DOWNLOAD](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[TEMPLATENAME] [varchar](200) NULL,
	[DESCRIPTION] [varchar](200) NULL,
	[PATH] [varchar](500) NULL,
	[IS_DELETE] [bit] NULL,
	[CREATED_DATE] [datetime] NULL,
	[CREATED_BY] [varchar](20) NULL,
	[MODIFIED_DATE] [datetime] NULL,
	[MODIFIED_BY] [varchar](20) NULL,
 CONSTRAINT [PK__T_TEMPLA__3214EC0735BE6AFF] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_TOUGH_AREA]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_TOUGH_AREA](
	[id] [bigint] IDENTITY(1,1) NOT NULL,
	[questionid] [bigint] NOT NULL,
	[T_SectionA] [varchar](50) NULL,
	[T_SectionB] [varchar](50) NULL,
	[T_SectionC] [varchar](50) NULL,
	[T_SectionD] [varchar](50) NULL,
	[flag] [bit] NULL,
 CONSTRAINT [PK_tbl_ToughArea] PRIMARY KEY CLUSTERED 
(
	[id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_UG_DEGREE]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_UG_DEGREE](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[UG_Degree_Name] [nvarchar](10) NULL,
 CONSTRAINT [PK_T_UG_DEGREE] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_UNIVERSITY_COUNSELLING]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_UNIVERSITY_COUNSELLING](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[Semester_Id] [bigint] NOT NULL,
	[DateOf_Counselling] [datetime] NULL,
	[Time] [nvarchar](50) NULL,
	[TopicsDiscussed] [nvarchar](280) NULL,
	[Action_Taken] [nvarchar](280) NULL,
	[Remarks] [text] NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](100) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](100) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_UniversityCounselling] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_UNIVERSITY_DOCUMENT]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_UNIVERSITY_DOCUMENT](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[StudentTable_Id] [bigint] NOT NULL,
	[Institution_Id] [bigint] NOT NULL,
	[Document_Id] [bigint] NOT NULL,
	[Description] [nvarchar](250) NULL,
	[Path] [nvarchar](250) NULL,
	[Status] [bit] NULL,
	[DeleteFlag] [bit] NULL,
	[Created_By] [nvarchar](150) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](150) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_UniversityDocument_1] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_USER_INSTITUTION]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_USER_INSTITUTION](
	[ID] [bigint] IDENTITY(1,1) NOT NULL,
	[Institution_Group_ID] [bigint] NULL,
	[Institution_ID] [bigint] NOT NULL,
	[User_ID] [bigint] NOT NULL,
	[Status] [bit] NULL,
	[Delete_Flag] [bit] NULL,
	[Created_By] [nvarchar](50) NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [nvarchar](50) NULL,
	[Modified_Date] [datetime] NULL,
 CONSTRAINT [PK_tbl_User_Institution] PRIMARY KEY CLUSTERED 
(
	[ID] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_USER_LOGIN_HISTORY]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[T_USER_LOGIN_HISTORY](
	[Login_History_id] [bigint] IDENTITY(1,1) NOT NULL,
	[UserId] [varchar](50) NULL,
	[LoginTime] [datetime] NULL,
	[LogoutTime] [datetime] NULL,
	[Remarks] [nvarchar](50) NULL,
 CONSTRAINT [PK_tbl_USER_LOGIN_HISTORY] PRIMARY KEY CLUSTERED 
(
	[Login_History_id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  Table [dbo].[T_WALLPAPER]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_WALLPAPER](
	[Id] [bigint] IDENTITY(1,1) NOT NULL,
	[Wallpaper_Name] [nvarchar](max) NULL,
	[Wallpaper_Path] [nvarchar](max) NULL,
	[Make_Default] [bit] NULL,
	[Data_Date] [datetime] NULL,
 CONSTRAINT [PK_Wallpaper] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[T_WIDGETS]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[T_WIDGETS](
	[id] [int] IDENTITY(1,1) NOT NULL,
	[widget_id] [nvarchar](100) NULL,
	[widget_name] [nvarchar](200) NULL,
	[user_id] [bigint] NULL,
	[widget_status] [bit] NOT NULL,
 CONSTRAINT [PK_Widgets] PRIMARY KEY CLUSTERED 
(
	[id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY]

GO
/****** Object:  Table [dbo].[tbl_Users]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE TABLE [dbo].[tbl_Users](
	[Id] [int] IDENTITY(1,1) NOT NULL,
	[UserId] [bigint] NULL,
	[UserType] [nvarchar](50) NULL,
	[UserTypeId] [bigint] NULL,
	[User_Name] [nvarchar](50) NULL,
	[Password] [nvarchar](50) NULL,
	[E_Mail] [nvarchar](50) NULL,
	[Profile_Picture_FilePath] [nvarchar](max) NULL,
	[Facebook_Id] [nvarchar](150) NULL,
	[GooglePlus_Id] [nvarchar](150) NULL,
	[Mobile_No] [nvarchar](15) NULL,
	[Twitter_Id] [nvarchar](150) NULL,
	[Last_Online] [datetime] NULL,
	[Is_Active] [bit] NULL,
	[Is_Deleted] [bit] NULL,
	[Created_By] [int] NULL,
	[Created_Date] [datetime] NULL,
	[Modified_By] [int] NULL,
	[Modified_Date] [datetime] NULL,
	[Locked] [bit] NULL,
	[Attempt] [int] NULL,
 CONSTRAINT [PK_Users] PRIMARY KEY CLUSTERED 
(
	[Id] ASC
)WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO
/****** Object:  Table [dbo].[test123]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[test123](
	[Type] [varchar](50) NULL,
	[TotalSales] [varchar](50) NULL
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO
/****** Object:  UserDefinedFunction [dbo].[fncAttendance]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE function [dbo].[fncAttendance](@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),
	@pSectionId NVARCHAR(50),
--	@pExamId NVARCHAR(50),
	@ExamFdate date,
	@ExamTdate date)
	returns
	table 
	as

	Return

SELECT     
--   Branch_Id, YearOfStudy_Id, Semester_Id, Section_Id,
  courseName,StudentId,StudentId_Rollno,Gender,Section_Id,Section_Name as[Section],RegNo,SNO,
 StudentName, CAST((PersentPeriod * 100) / (PersentPeriod + AbsentPeriod) 
                         AS float) AS PercentageOfAttendance
FROM  (SELECT DISTINCT 
       StudentName,courseName,StudentId,StudentId_Rollno,Gender,Section_Id,Section_Name,RegNo,SNO,
	   SUM(CASE WHEN TempTab.Attendance_Status = 'P' THEN 1 ELSE 0 END) AS PersentPeriod, 
       SUM(CASE WHEN TempTab.Attendance_Status = 'AB' THEN 1 ELSE 0 END) AS AbsentPeriod
       FROM(SELECT DISTINCT 
            AE.Branch_Id, AE.YearOfStudy_Id, AE.Semester_Id, AE.Section_Id,SM.Section_Name, STU.StudentName +' '+STU.Initial as StudentName,stu.Id as StudentId,stu.StudentId_Rollno,stu.Gender,stu.UniversityRegNo as RegNo,stu.SNo as SNO,CM.Course_Code + '-'+CM.Course_ShortCode AS courseName, 
            AE.Course_Id, AE.Attendance_Status, AE.Attendance_Date, AE.Period
            FROM  dbo.T_ATTENDANCEENTRY AS AE (NOLOCK)
			INNER JOIN  dbo.T_CFG_STUDENT_PERSONAL_DETAIL AS STU ON AE.StudentTable_Id = STU.Id 
			INNER JOIN  dbo.T_CFG_COURSE_MASTER AS CM ON AE.Course_Id = CM.Course_Id
			join T_CFG_SECTION_MASTER AS SM on STU.Section_Id=SM.ID
            WHERE (AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId and AE.Section_Id=@pSectionId and CONVERT(date,AE.Attendance_Date,103) between CONVERT(date,@ExamFdate,103) and CONVERT(date,@ExamTdate,103) )) AS TempTab
GROUP BY Course_Id, courseName, StudentName,StudentId,RegNo,SNO,StudentId_Rollno,Gender,Section_Id,Section_Name) AS finalTable
GO
/****** Object:  UserDefinedFunction [dbo].[fncAttendanceAll]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE function [dbo].[fncAttendanceAll](@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),
	@pSectionId NVARCHAR(50),
--	@pExamId NVARCHAR(50),
	@ExamFdate date,
	@ExamTdate date)
	returns
	table 
	as

	Return

SELECT     
--   Branch_Id, YearOfStudy_Id, Semester_Id, Section_Id,
  courseName,StudentId,StudentId_Rollno,Gender,Section_Id,RegNo,SNO,
 StudentName,Section_Name as[Section], CAST((PersentPeriod * 100) / (PersentPeriod + AbsentPeriod) 
                         AS float) AS PercentageOfAttendance
FROM(SELECT DISTINCT 
StudentName,courseName,StudentId,StudentId_Rollno,Gender,Section_Id,Section_Name ,RegNo,SNO,
SUM(CASE WHEN TempTab.Attendance_Status = 'P' THEN 1 ELSE 0 END) AS PersentPeriod, 
SUM(CASE WHEN TempTab.Attendance_Status = 'AB' THEN 1 ELSE 0 END) AS AbsentPeriod
FROM (SELECT DISTINCT
 AE.Branch_Id, AE.YearOfStudy_Id, AE.Semester_Id, AE.Section_Id,SM.Section_Name, STU.StudentName+' '+STU.Initial as StudentName,stu.Id as StudentId,stu.StudentId_Rollno,stu.Gender,stu.UniversityRegNo as RegNo,stu.SNo as SNO,CM.Course_Code+'-'+ CM.Course_ShortCode AS courseName, 
AE.Course_Id, AE.Attendance_Status, AE.Attendance_Date, AE.Period
 FROM dbo.T_ATTENDANCEENTRY AS AE (NOLOCK)
 INNER JOIN
  dbo.T_CFG_STUDENT_PERSONAL_DETAIL AS STU ON AE.StudentTable_Id = STU.Id
   INNER JOIN
    dbo.T_CFG_COURSE_MASTER AS CM ON AE.Course_Id = CM.Course_Id
	join T_CFG_SECTION_MASTER AS SM on STU.Section_Id=SM.ID
  WHERE (AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId  and CONVERT(date,AE.Attendance_Date,103) between CONVERT(date,@ExamFdate,103) and CONVERT(date,@ExamTdate,103) )) AS TempTab
GROUP BY Course_Id, courseName, StudentName,StudentId,RegNo,SNO,StudentId_Rollno,Gender,Section_Id,Section_Name
) AS finalTable
GO
/****** Object:  UserDefinedFunction [dbo].[fncAttendanceAvg]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE function [dbo].[fncAttendanceAvg](@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),
	@pSectionId NVARCHAR(50),
--	@pExamId NVARCHAR(50),
	@ExamFdate date,
	@ExamTdate date)
returns
	table 
	as

	Return
select StudentTable_Id,temp.workingDates as numberOfWorkingDays from(select ae.StudentTable_Id, ae.Attendance_Date, count(ae.Attendance_Date) as workingDates  from T_ATTENDANCEENTRY AE (NOLOCK)
where ae.Attendance_Status='P' and AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId and AE.Section_Id=@pSectionId and CONVERT(date,AE.Attendance_Date,103) between CONVERT(date,@ExamFdate,103)  and CONVERT(date,@ExamTdate,103)
group by ae.Attendance_Date,ae.StudentTable_Id) as temp
GO
/****** Object:  UserDefinedFunction [dbo].[fncAttendanceAvgAll]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE function [dbo].[fncAttendanceAvgAll](@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),
	@pSectionId NVARCHAR(50),
--	@pExamId NVARCHAR(50),
	@ExamFdate date,
	@ExamTdate date)
returns
	table 
	as

	Return
select StudentTable_Id,temp.workingDates as numberOfWorkingDays from(select ae.StudentTable_Id, ae.Attendance_Date, count(ae.Attendance_Date) as workingDates  from T_ATTENDANCEENTRY AE (NOLOCK)
where ae.Attendance_Status='P' and AE.Branch_Id=@pBranchId and AE.YearOfStudy_Id=@pYearOfStudyId and AE.Semester_Id=@pSemesterId  and CONVERT(date,AE.Attendance_Date,103) between CONVERT(date,@ExamFdate,103)  and CONVERT(date,@ExamTdate,103)
group by ae.Attendance_Date,ae.StudentTable_Id) as temp
GO
/****** Object:  UserDefinedFunction [dbo].[fncAttendanceReport]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
--select * from fncAttendance(8,6,12,3,'2014-03-10','2014-03-10')
CREATE function [dbo].[fncAttendanceReport](
	@pfromdate NVARCHAR(50),
	@pTodate NVARCHAR(50),
	@pBranch NVARCHAR(50),
	@pYear NVARCHAR(50))

	returns
	table 
	as

	Return
select    a.StudentTable_Id,c.StudentName+' '+c.Initial as StudentName,c.StudentId_Rollno, c.UniversityRegNo, (y.YearOfStudy_Name+' | '+s.Section_Name+' | '+sem.Semester_Name) as YearOfStudy_Id,(convert(nvarchar(10),count(distinct(Attendance_Date)))+' - Leave ') As Leavecount, count(distinct(Attendance_Date)) as [lCount]  from  [dbo].[T_ATTENDANCEENTRY] As a 
join T_CFG_STUDENT_PERSONAL_DETAIL As c on  a.StudentTable_Id= c.Id 
join T_CFG_YEAROFSTUDY_MASTER As y on c.YearOfStudy_Id = y.YearOfStudy_Id 
join T_CFG_SECTION_MASTER As s on c.Section_Id = s.ID  
join T_CFG_SEMESTER sem on c.Semester_Id = sem.Semester_Id join T_CFG_ACADEMIC_YEAR  Acd  on c.AcademicYear_Id = Acd.ID
where a.Attendance_Date >=@pfromdate  and  a.Attendance_Date <= @pTodate and a.Attendance_Status = 'AB' and 
c.Delete_Flag ='false' and  c.TC_ISSUED = 'false' and Acd.Academic_year_Status = 'true' and c.Branch_Id = @pBranch	and y.YearOfStudy_Id=@pYear

	group by  a.StudentTable_Id,c.StudentName,c.Initial,c.StudentId_Rollno, c.UniversityRegNo, y.YearOfStudy_Name,s.Section_Name,sem.Semester_Name
	-- having count(distinct(Attendance_Date)) > 3

	union

	select   a.Student_Id,c.StudentName+' '+c.Initial as StudentName,c.StudentId_Rollno, c.UniversityRegNo, (y.YearOfStudy_Name+' | '+s.Section_Name+' | '+sem.Semester_Name) as YearOfStudy_Id,(convert(nvarchar(10),count(distinct(From_Date)))+' - OD ') As Leavecount , count(distinct(From_Date)) as [lCount] from T_OD_DETAILS a 
join T_CFG_STUDENT_PERSONAL_DETAIL c on a.Student_Id = c.Id 
join T_CFG_YEAROFSTUDY_MASTER y on c.YearOfStudy_Id = y.YearOfStudy_Id  
join T_CFG_SECTION_MASTER s on c.Section_Id = s.ID
join T_CFG_SEMESTER sem on c.Semester_Id = sem.Semester_Id join T_CFG_STUDENT_PARENT_DETAIL m on c.Id = m.StudentTable_Id join T_CFG_ACADEMIC_YEAR Acd on c.AcademicYear_Id = Acd.ID
where a.From_Date >=  @pfromdate and a.From_Date <= @pTodate and a.OD_Approve = 'true' and c.Delete_Flag = 'false'and  c.TC_ISSUED = 'false' and Acd.Academic_year_Status = 'true' and c.Branch_Id = @pBranch
	   --and c.StudentId_Rollno ='NAIT4A52'
and y.YearOfStudy_Id=@pYear
group by  a.Student_Id,c.StudentName,c.Initial,c.StudentId_Rollno, c.UniversityRegNo, y.YearOfStudy_Name,s.Section_Name,sem.Semester_Name	

GO
/****** Object:  UserDefinedFunction [dbo].[fncAttendanceReportNEW]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE function [dbo].[fncAttendanceReportNEW](
	@pfromdate NVARCHAR(50),
	@pTodate NVARCHAR(50),
	@pBranch NVARCHAR(50),
	@pYear NVARCHAR(50))

	returns
	table 
	as
	
	Return

	

select    a.StudentTable_Id,c.StudentName+' '+c.Initial as StudentName,c.StudentId_Rollno, c.UniversityRegNo, (br.branch_code +' | '+ y.YearOfStudy_Name+' | '+s.Section_Name+' | '+sem.Semester_Name) as YearOfStudy_Id,(convert(nvarchar(10),count(distinct(Attendance_Date)))+' - Leave ') As Leavecount, count(distinct(Attendance_Date)) as [lCount]  from  [dbo].[T_ATTENDANCEENTRY] As a 
join T_CFG_STUDENT_PERSONAL_DETAIL As c on  a.StudentTable_Id= c.Id 
join T_CFG_BRANCH br on c.Branch_Id = br.Branch_Id
join T_CFG_YEAROFSTUDY_MASTER As y on c.YearOfStudy_Id = y.YearOfStudy_Id 
join T_CFG_SECTION_MASTER As s on c.Section_Id = s.ID  
join T_CFG_SEMESTER sem on c.Semester_Id = sem.Semester_Id
 join T_CFG_ACADEMIC_YEAR  Acd  on c.AcademicYear_Id = Acd.ID
where a.Attendance_Date >=@pfromdate  and  a.Attendance_Date <= @pTodate and a.Attendance_Status = 'AB' and 
c.Delete_Flag ='false' and  c.TC_ISSUED = 'false' and Acd.Academic_year_Status = 'true' and c.Branch_Id in (select id from CSVToTable(@pBranch))	and y.YearOfStudy_Id=@pYear

	group by  a.StudentTable_Id,c.StudentName,c.Initial,c.StudentId_Rollno, c.UniversityRegNo,br.branch_code ,y.YearOfStudy_Name,s.Section_Name,sem.Semester_Name
	-- having count(distinct(Attendance_Date)) > 3

	union

	select   b.StudentTable_Id,c.StudentName+' '+c.Initial as StudentName,c.StudentId_Rollno, c.UniversityRegNo, (br.branch_code +' | '+ y.YearOfStudy_Name+' | '+s.Section_Name+' | '+sem.Semester_Name) as YearOfStudy_Id,(convert(nvarchar(10),DATEDIFF(day, DATEADD(day, -1, a.From_Date), a.To_Date))+' - OD ') As Leavecount , DATEDIFF(day, DATEADD(day, -1, a.From_Date), a.To_Date) as [lCount] from T_OD_DETAILS_HDR a 
	join T_OD_DETAILS_DTL b on a.ODId=b.ODHdr_Id
join T_CFG_STUDENT_PERSONAL_DETAIL c on b.StudentTable_Id = c.Id 
join T_CFG_BRANCH br on c.Branch_Id = br.Branch_Id
join T_CFG_YEAROFSTUDY_MASTER y on c.YearOfStudy_Id = y.YearOfStudy_Id  
join T_CFG_SECTION_MASTER s on c.Section_Id = s.ID
join T_CFG_SEMESTER sem on c.Semester_Id = sem.Semester_Id 
join T_CFG_STUDENT_PARENT_DETAIL m on c.Id = m.StudentTable_Id 
join T_CFG_ACADEMIC_YEAR Acd on c.AcademicYear_Id = Acd.ID
where  (a.From_Date between @pfromdate and @pTodate)  and (a.To_Date between @pfromdate and @pTodate) and b.OD_Approve = 1 and b.Delete_Flag=0 and c.Delete_Flag = 0 and  c.TC_ISSUED = 0 and Acd.Academic_year_Status = 1 and c.Branch_Id in (select id from CSVToTable(@pBranch))
	   --and c.StudentId_Rollno ='NAIT4A52'
and y.YearOfStudy_Id=@pYear
group by b.ODHdr_Id, b.StudentTable_Id,c.StudentName,c.Initial,c.StudentId_Rollno, c.UniversityRegNo,br.branch_code, y.YearOfStudy_Name,s.Section_Name,sem.Semester_Name,a.From_Date,a.To_Date



--select * from fncAttendanceReportNEW('2014-07-01','2014-10-31','8,1',2) 

--select * from CSVToTable('8,1,2')







GO
/****** Object:  UserDefinedFunction [dbo].[fncConsecutiveLeaveAttendanceReport]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE function [dbo].[fncConsecutiveLeaveAttendanceReport](
	@pfromdate NVARCHAR(50),
	@pTodate NVARCHAR(50),
	@pBranch NVARCHAR(50),
	@pYear NVARCHAR(50))

	returns
	table 
	as

	Return
select    a.StudentTable_Id,c.StudentName+''+c.Initial as StudentName,c.StudentId_Rollno, c.UniversityRegNo, Attendance_Date, 
(Br.Branch_Code+' | '+y.YearOfStudy_Name+' | '+s.Section_Name+' | '+sem.Semester_Name) as YearOfStudy_Id,
DiffCode = DATEDIFF(DAY, 0, Attendance_Date) - ROW_NUMBER() OVER(PARTITION BY StudentTable_Id ORDER BY Attendance_Date) 
from  [dbo].[T_ATTENDANCEENTRY] As a 
join T_CFG_STUDENT_PERSONAL_DETAIL As c on  a.StudentTable_Id= c.Id 
join T_CFG_BRANCH Br on c.Branch_Id=Br.Branch_Id
join T_CFG_YEAROFSTUDY_MASTER As y on c.YearOfStudy_Id = y.YearOfStudy_Id 
join T_CFG_SECTION_MASTER As s on c.Section_Id = s.ID  
join T_CFG_SEMESTER sem on c.Semester_Id = sem.Semester_Id join T_CFG_ACADEMIC_YEAR  Acd  on c.AcademicYear_Id = Acd.ID
where a.Attendance_Date >=@pfromdate  and  a.Attendance_Date <= @pTodate and a.Attendance_Status = 'AB' and 
c.Delete_Flag ='false' and  c.TC_ISSUED = 'false' and Acd.Academic_year_Status = 'true' and c.Branch_Id in (select id from CSVToTable(@pBranch)) and y.YearOfStudy_Id=@pYear

group by  a.StudentTable_Id,c.StudentName,c.Initial,c.StudentId_Rollno, c.UniversityRegNo, Attendance_Date,Br.Branch_Code, y.YearOfStudy_Name,s.Section_Name,sem.Semester_Name
-- having count(distinct(Attendance_Date)) > 3	



--select * from fncConsecutiveLeaveAttendanceReport('2014-06-01','2014-08-24','8,1',3)
GO
/****** Object:  UserDefinedFunction [dbo].[fnGetClassAverage]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
create function [dbo].[fnGetClassAverage]()
returns table
as
return
SELECT Course_Name AS [NAME OF THE SUBJECT], Course_Code AS [SUBJECT CODE], [UNIT TEST I],[UNIT TEST II],[MODEL EXAM] FROM
(SELECT CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name,
round(SUM(CAST(ME.Marks AS float))/38,2) as [retestMarksPivot]
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 --WHERE ME.StudentTable_Id = 1135
		 and ME.Branch_Id=8 and ME.YearOfStudy_Id = 3 and ME.Semester_Id = 6 and me.Section_Id=1
		 group by CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name
		 ) AS SRC
		 PIVOT (MAX([retestMarksPivot]) FOR Exam_Name IN ([UNIT TEST I],[UNIT TEST II],[MODEL EXAM])) AS PVT
GO
/****** Object:  UserDefinedFunction [dbo].[fnGetStudentMark]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
create function [dbo].[fnGetStudentMark]()
returns table
as
return
SELECT Course_Name AS [NAME OF THE SUBJECT], Course_Code AS [SUBJECT CODE], [UNIT TEST I],[UNIT TEST II],[MODEL EXAM] FROM
(SELECT ME.StudentTable_Id, CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name,
 case when (me.Is_Absent=1 and (me.marks='0' or me.Marks='00' or me.Marks='000')) then 'AB'  when (me.Is_OD=1 and (me.marks='0' or me.Marks='00' or me.Marks='000')) then 'OD' else ME.Marks end as [marksPivot]
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = 1135
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = 3 and ME.Semester_Id = 6) AS SRC
		 PIVOT (MAX([marksPivot]) FOR Exam_Name IN ([UNIT TEST I],[UNIT TEST II],[MODEL EXAM])) AS PVT

GO
/****** Object:  UserDefinedFunction [dbo].[fnGetStudentRetestMark]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
create function [dbo].[fnGetStudentRetestMark]()
returns table
as
return
SELECT Course_Name AS [NAME OF THE SUBJECT], Course_Code AS [SUBJECT CODE], [UNIT TEST I],[UNIT TEST II],[MODEL EXAM] FROM
(SELECT ME.StudentTable_Id, CM.Course_Name, CM.Course_ShortCode, CM.Course_Code, EM.Exam_Name,
 case when (me.Is_Absent_Retest=1 and (me.RetestMarks='0' or me.RetestMarks='00' or me.RetestMarks='000')) then 'AB' else ME.RetestMarks end as [retestMarksPivot]
		 FROM T_MARK_ENTRY ME (NOLOCK)
		 JOIN T_CFG_COURSE_MASTER CM ON ME.Course_Id = CM.Course_Id
		 JOIN T_CFG_EXAM_MASTER EM ON ME.Exam_Id = EM.Exam_Id
		 JOIN T_CFG_STUDENT_PERSONAL_DETAIL S ON ME.StudentTable_Id = S.Id
		 WHERE ME.StudentTable_Id = 1135
		 and ME.Branch_Id=S.Branch_Id and ME.YearOfStudy_Id = 3 and ME.Semester_Id = 6) AS SRC
		 PIVOT (MAX([retestMarksPivot]) FOR Exam_Name IN ([UNIT TEST I],[UNIT TEST II],[MODEL EXAM])) AS PVT
GO
/****** Object:  UserDefinedFunction [dbo].[fnPoorOverAll]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE FUNCTION [dbo].[fnPoorOverAll]
(
@pInsitutionId nvarchar(50), @pBranchId nvarchar(50),
 @pYearofStudyId nvarchar(50),@pSemesterId nvarchar(50),
 @pExamId nvarchar(50),@pNoofToppers nvarchar(50),
 @pAcYr nvarchar(10)
)

RETURNS TABLE 
AS
RETURN 
(
	with marksInfo(BranchName,StuTabId,[Student Id],UniversityRegNo,StudentName,Section_Name,Course_Code,Course_Name,Marks)
as
(

 select brnch.Branch_Name as BranchName,stu.Id [StuTabId],stu.StudentId_Rollno as [Student Id],stu.UniversityRegNo,stu.StudentName+' '+stu.Initial as StudentName,sec.Section_Name,cou.Course_Code,cou.Course_Name,mark.Marks
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_MARK_ENTRY mark (NOLOCK) on stu.Id = mark.StudentTable_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
join T_CFG_COURSE_MASTER cou on mark.Course_Id=cou.Course_Id
join T_CFG_BRANCH brnch on stu.Branch_Id= brnch.Branch_Id
where mark.Branch_Id=cast(@pBranchId as bigint) 
and mark.YearOfStudy_Id=cast(@pYearofStudyId as bigint) and mark.Semester_Id=cast(@pSemesterId as bigint) and mark.Exam_Id=@pExamId 
and stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0  and mark.AcademicYear_Id=cast(@pAcYr as bigint)
)
,
TotalInfo(StuTabId,UniversityRegNo,StudentName,Total,[rank_No])
As
(


 select stu.Id [StuTabId],stu.UniversityRegNo,stu.StudentName+' '+stu.Initial as StudentName,sum(cast(mark.marks as int)) as Total,
DENSE_RANK() over (order by sum(cast(mark.marks as int))asc) as [rank_No] 
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_MARK_ENTRY mark (NOLOCK) on stu.Id = mark.StudentTable_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
join T_CFG_COURSE_MASTER cou on mark.Course_Id=cou.Course_Id
where mark.Branch_Id=cast(@pBranchId as bigint) 
and mark.YearOfStudy_Id=cast(@pYearofStudyId as bigint) and mark.Semester_Id=cast(@pSemesterId as bigint) and mark.Exam_Id=@pExamId 
and stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0  and mark.AcademicYear_Id=cast(@pAcYr as bigint)
group by stu.UniversityRegNo,stu.StudentName,stu.Initial,stu.Id
)



select a.BranchName,a.[Student Id], a.UniversityRegNo,a.StudentName,a.Section_Name,a.Course_Code,a.Marks,b.Total,b.rank_No
from marksInfo a
join TotalInfo b on a.StuTabId=b.StuTabId

where b.[rank_No]<=@pNoofToppers  
)

GO
/****** Object:  UserDefinedFunction [dbo].[fnPoorWithSection]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE FUNCTION [dbo].[fnPoorWithSection]
(
@pInsitutionId nvarchar(50), @pBranchId nvarchar(50),
 @pYearofStudyId nvarchar(50),@pSemesterId nvarchar(50),
 @pExamId nvarchar(50),@pNoofToppers nvarchar(50),
 @pAcYr nvarchar(10)
)

RETURNS TABLE 
AS
RETURN 
(
	with marksInfo(BranchName,[Student Id],UniversityRegNo,StudentName,Section_Name,Course_Code,Course_Name,Marks)
as
(

 select brnch.Branch_Name as BranchName,stu.StudentId_Rollno as [Student Id],stu.UniversityRegNo,stu.StudentName+' '+stu.Initial as StudentName,sec.Section_Name,cou.Course_Code,cou.Course_Name,mark.Marks
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_MARK_ENTRY mark (NOLOCK) on stu.Id = mark.StudentTable_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
join T_CFG_COURSE_MASTER cou on mark.Course_Id=cou.Course_Id
join T_CFG_BRANCH brnch on stu.Branch_Id= brnch.Branch_Id
where mark.Branch_Id=cast(@pBranchId as bigint) 
and mark.YearOfStudy_Id=cast(@pYearofStudyId as bigint) and mark.Semester_Id=cast(@pSemesterId as bigint) and mark.Exam_Id=@pExamId 
and stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0  and mark.AcademicYear_Id=cast(@pAcYr as bigint)
)
,
TotalInfo(UniversityRegNo,StudentName,Section_Name,Total,[rank_No])
As
(


 select stu.UniversityRegNo,stu.StudentName+' '+stu.Initial as StudentName,sec.Section_Name,sum(cast(mark.marks as int)) as Total,
DENSE_RANK() over (partition by stu.section_id order by sum(cast(mark.marks as int))) as [rank_No] 
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_MARK_ENTRY mark (NOLOCK) on stu.Id = mark.StudentTable_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
join T_CFG_COURSE_MASTER cou on mark.Course_Id=cou.Course_Id
where mark.Branch_Id=cast(@pBranchId as bigint) 
and mark.YearOfStudy_Id=cast(@pYearofStudyId as bigint) and mark.Semester_Id=cast(@pSemesterId as bigint) and mark.Exam_Id=@pExamId 
and stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0  and mark.AcademicYear_Id=cast(@pAcYr as bigint)
group by stu.UniversityRegNo,stu.StudentName,stu.Initial,sec.Section_Name,stu.Section_Id
)



select BranchName,a.[Student Id], a.UniversityRegNo,a.StudentName,a.Section_Name,a.Course_Code,a.Marks,b.Total,b.rank_No from marksInfo a
join TotalInfo b on a.UniversityRegNo=b.UniversityRegNo
where b.[rank_No]<=@pNoofToppers 
)
GO
/****** Object:  UserDefinedFunction [dbo].[fnToppersOverAll]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE FUNCTION [dbo].[fnToppersOverAll]
(
@pInsitutionId nvarchar(50), @pBranchId nvarchar(50),
 @pYearofStudyId nvarchar(50),@pSemesterId nvarchar(50),
 @pExamId nvarchar(50),@pNoofToppers nvarchar(50),@pAcYr nvarchar(10)
)

RETURNS TABLE 
AS
RETURN 
(
	with marksInfo(BranchName,StuTabId,[Student Id],UniversityRegNo,StudentName,Section_Name,Course_Code,Course_Name,Marks)
as
(

 select brnch.Branch_Name as BranchName,stu.Id [StuTabId],stu.StudentId_Rollno as [Student Id],stu.UniversityRegNo,stu.StudentName+' '+stu.Initial as StudentName,sec.Section_Name,cou.Course_Code,cou.Course_Name,mark.Marks
from T_MARK_ENTRY mark (NOLOCK)
join  T_CFG_STUDENT_PERSONAL_DETAIL stu on stu.Id = mark.StudentTable_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
join T_CFG_COURSE_MASTER cou on mark.Course_Id=cou.Course_Id
join T_CFG_BRANCH brnch on stu.Branch_Id= brnch.Branch_Id
where mark.Branch_Id=cast(@pBranchId as bigint) 
and mark.YearOfStudy_Id=cast(@pYearofStudyId as bigint) and mark.Semester_Id=cast(@pSemesterId as bigint) and mark.Exam_Id=@pExamId 
and stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0  and mark.AcademicYear_Id=cast(@pAcYr as bigint)
)
,
TotalInfo(StuTabId,UniversityRegNo,StudentName,Total,[rank_No])
As
(


 select stu.Id [StuTabId],stu.UniversityRegNo,stu.StudentName+' '+stu.Initial as StudentName,sum(cast(mark.marks as int)) as Total,
DENSE_RANK() over (order by sum(cast(mark.marks as int)) desc) as [rank_No] 
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_MARK_ENTRY mark (NOLOCK) on stu.Id = mark.StudentTable_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
join T_CFG_COURSE_MASTER cou on mark.Course_Id=cou.Course_Id
where mark.Institution_Id=cast(@pInsitutionId as bigint) and mark.Branch_Id=cast(@pBranchId as bigint) 
and mark.YearOfStudy_Id=cast(@pYearofStudyId as bigint) and mark.Semester_Id=cast(@pSemesterId as bigint) and mark.Exam_Id=cast(@pExamId as bigint)
and stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0 and mark.AcademicYear_Id=cast(@pAcYr as bigint)
group by stu.UniversityRegNo,stu.StudentName,stu.Initial,stu.Id
)



select a.BranchName,a.[Student Id], a.UniversityRegNo,a.StudentName,a.Section_Name,a.Course_Code,a.Marks,b.Total,b.rank_No
from marksInfo a
join TotalInfo b on a.StuTabId=b.StuTabId

where b.[rank_No]<=@pNoofToppers 
)


--select * from fnToppersOverAll(1,2,2,3,1,3,4)
GO
/****** Object:  UserDefinedFunction [dbo].[fnToppersWithSection]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE FUNCTION [dbo].[fnToppersWithSection]
(
@pInsitutionId nvarchar(50), @pBranchId nvarchar(50),
 @pYearofStudyId nvarchar(50),@pSemesterId nvarchar(50),
 @pExamId nvarchar(50),@pNoofToppers nvarchar(50),
 @pAcYr nvarchar(10)
)

RETURNS TABLE 
AS
RETURN 
(
	with marksInfo(BranchName,[Student Id],UniversityRegNo,StudentName,Section_Name,Course_Code,Course_Name,Marks)
as
(

 select brnch.Branch_Name as BranchName,stu.StudentId_Rollno as [Student Id],stu.UniversityRegNo,stu.StudentName+' '+stu.Initial as StudentName,sec.Section_Name,cou.Course_Code,cou.Course_Name,mark.Marks
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_MARK_ENTRY mark (NOLOCK) on stu.Id = mark.StudentTable_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
join T_CFG_COURSE_MASTER cou on mark.Course_Id=cou.Course_Id
join T_CFG_BRANCH brnch on stu.Branch_Id= brnch.Branch_Id
where  mark.Branch_Id=cast(@pBranchId as bigint) 
and mark.YearOfStudy_Id=cast(@pYearofStudyId as bigint) and mark.Semester_Id=cast(@pSemesterId as bigint) and mark.Exam_Id=@pExamId 
and stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0 and mark.AcademicYear_Id=cast(@pAcYr as bigint)

 
)
,
TotalInfo(UniversityRegNo,StudentName,Section_Name,Total,[rank_No])
As
(


 select stu.UniversityRegNo,stu.StudentName+' '+stu.Initial as StudentName,sec.Section_Name,sum(cast(mark.marks as int)) as Total,
DENSE_RANK() over (partition by stu.section_id order by sum(cast(mark.marks as int)) desc) as [rank_No] 
from T_CFG_STUDENT_PERSONAL_DETAIL stu
join T_MARK_ENTRY mark (NOLOCK) on stu.Id = mark.StudentTable_Id
join T_CFG_SECTION_MASTER sec on stu.Section_Id=sec.ID
join T_CFG_COURSE_MASTER cou on mark.Course_Id=cou.Course_Id
where  mark.Branch_Id=cast(@pBranchId as bigint) 
and mark.YearOfStudy_Id=cast(@pYearofStudyId as bigint) and mark.Semester_Id=cast(@pSemesterId as bigint) and mark.Exam_Id=cast(@pExamId as bigint)
and stu.Delete_Flag=0 and stu.Status=1 and stu.TC_ISSUED=0 and mark.AcademicYear_Id=cast(@pAcYr as bigint)  
group by stu.UniversityRegNo,stu.StudentName,stu.Initial,sec.Section_Name,stu.Section_Id
)



select BranchName,a.[Student Id], a.UniversityRegNo,a.StudentName,a.Section_Name,a.Course_Code,a.Marks,b.Total,b.rank_No from marksInfo a
join TotalInfo b on a.UniversityRegNo=b.UniversityRegNo
where b.[rank_No]<=@pNoofToppers 
)


--select * from fnToppersWithSection(1,8,2,3,1,3,3)
GO
/****** Object:  UserDefinedFunction [dbo].[fnTotalAndAttendance]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:		<Author,,Name>
-- Create date: <Create Date,,>
-- Description:	<Description,,>
-- =============================================
CREATE FUNCTION [dbo].[fnTotalAndAttendance]
(	
	-- Add the parameters for the function here
	@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),	
	@pExamId NVARCHAR(50),
	@pFromDt datetime,
	@pToDt datetime,
	@pAcYr bigint
)
RETURNS TABLE 
AS
RETURN 
(
	-- Add the SELECT statement with parameter references here

	with TotalInfo([StuTabId],Gender,SNO,StudentId_Rollno,UniversityRegNo,StudentName,Section,Total,[rank_No])
	as
	(
	Select sp.Id as[StuTabId], SP.Gender,SP.SNo, SP.StudentId_Rollno, SP.UniversityRegNo,SP.StudentName+' '+SP.Initial as StudentName,
	SC.Section_Name as Section , 
	sum(cast(ME.marks as int)) as Total,
	DENSE_RANK() over (partition by SP.section_id order by sum(cast(ME.marks as int)) desc) as [rank_No] 	
	from T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	join T_CFG_SECTION_MASTER SC on SP.Section_Id=SC.id
	where ME.AcademicYear_Id=@pAcYr and ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and 
		  ME.Semester_Id=@pSemesterId and
		  ME.Exam_Id=@pExamId and SP.Delete_Flag=0 and SP.Status=1 and SP.TC_ISSUED=0 
	group by sp.Id,SP.Gender,SP.SNo, SP.StudentId_Rollno, SP.UniversityRegNo,SP.StudentName,SP.Initial,SC.Section_Name,SP.section_id
	),


	--select * from T_CFG_STUDENT_PERSONAL_DETAIL 
	--where Branch_Id=8 and YearOfStudy_Id=2 and Semester_Id=3 and Section_Id=3 and Delete_Flag=0 and status=1 and TC_ISSUED=0 and AcademicYear_Id=3

	AttendanceInfo([StuTabId],Gender,SNO,StudentId_Rollno,UniversityRegNo,StudentName,Section,[Attendance Percentage])
	as
	(
	select sp.Id as[StuTabId],SP.Gender,SP.SNo, SP.StudentId_Rollno, SP.UniversityRegNo,SP.StudentName+' '+SP.Initial as StudentName,
	SC.Section_Name as Section,
	cast((cast(sum(case when att.Attendance_Status='P' then 1 else 0 end)*100 as decimal(10,2))/count(att.Attendance_Status)) as decimal(10,2)) as [Attendance Percentage]
	from T_ATTENDANCEENTRY att (NOLOCK)
	join T_CFG_STUDENT_PERSONAL_DETAIL SP on att.StudentTable_Id = SP.Id
	join T_CFG_SECTION_MASTER SC on SP.Section_Id=SC.id
	where  SP.Branch_Id=@pBranchId and SP.YearOfStudy_Id=@pYearOfStudyId and SP.Semester_Id=@pSemesterId 
	and SP.Delete_Flag=0 and SP.status=1 and SP.TC_ISSUED=0 
	and att.Attendance_Date between @pFromDt and @pToDt
	group by sp.Id,SP.Gender,SP.SNo, SP.StudentId_Rollno, SP.UniversityRegNo,SP.StudentName,SP.Initial,
	SC.Section_Name
	)



	select a.StuTabId,a.StudentName,a.Total,a.rank_No,b.[Attendance Percentage] from TotalInfo a
	left outer join  AttendanceInfo b on a.StudentId_Rollno= b.StudentId_Rollno

	
)


GO
/****** Object:  UserDefinedFunction [dbo].[fnTotalAndAttendanceWithSection]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE FUNCTION [dbo].[fnTotalAndAttendanceWithSection]
(	
	-- Add the parameters for the function here
	@pBranchId NVARCHAR(50),
	@pYearOfStudyId NVARCHAR(50),
	@pSemesterId NVARCHAR(50),
	@pSectionId NVARCHAR(50),
	@pExamId NVARCHAR(50),
	@pFromDt datetime,
	@pToDt datetime,
	@pAcYr bigint
)
RETURNS TABLE 
AS
RETURN 
(
	-- Add the SELECT statement with parameter references here

	with TotalInfo([StuTabId],Gender,SNO,StudentId_Rollno,UniversityRegNo,StudentName,Section,Total,[rank_No])
	as
	(
	Select sp.Id as[StuTabId], SP.Gender,SP.SNo, SP.StudentId_Rollno, SP.UniversityRegNo,SP.StudentName+' '+SP.Initial as StudentName,
	SC.Section_Name as Section , 
	sum(cast(ME.marks as int)) as Total,
	DENSE_RANK() over (partition by SP.section_id order by sum(cast(ME.marks as int)) desc) as [rank_No] 	
	from T_CFG_STUDENT_PERSONAL_DETAIL SP 
	join T_MARK_ENTRY ME (NOLOCK) on SP.Id=ME.StudentTable_Id
	join T_CFG_COURSE_MASTER CM on ME.Course_Id=CM.Course_Id
	join T_CFG_SECTION_MASTER SC on SP.Section_Id=SC.id
	where ME.AcademicYear_Id=@pAcYr and ME.Branch_Id=@pBranchId and ME.YearOfStudy_Id=@pYearOfStudyId and ME.Section_Id =@pSectionId and
		  ME.Semester_Id=@pSemesterId and
		  ME.Exam_Id=@pExamId and SP.Delete_Flag=0 and SP.Status=1 and SP.TC_ISSUED=0 
	group by sp.Id,SP.Gender,SP.SNo, SP.StudentId_Rollno, SP.UniversityRegNo,SP.StudentName,SP.Initial,SC.Section_Name,SP.section_id
	),


	--select * from T_CFG_STUDENT_PERSONAL_DETAIL 
	--where Branch_Id=8 and YearOfStudy_Id=2 and Semester_Id=3 and Section_Id=3 and Delete_Flag=0 and status=1 and TC_ISSUED=0 and AcademicYear_Id=3

	AttendanceInfo([StuTabId],Gender,SNO,StudentId_Rollno,UniversityRegNo,StudentName,Section,[Attendance Percentage])
	as
	(
	select sp.Id as[StuTabId],SP.Gender,SP.SNo, SP.StudentId_Rollno, SP.UniversityRegNo,SP.StudentName+' '+SP.Initial as StudentName,
	SC.Section_Name as Section,
	cast((cast(sum(case when att.Attendance_Status='P' then 1 else 0 end)*100 as decimal(10,2))/count(att.Attendance_Status)) as decimal(10,2)) as [Attendance Percentage]
	from T_ATTENDANCEENTRY att (NOLOCK)
	join T_CFG_STUDENT_PERSONAL_DETAIL SP on att.StudentTable_Id = SP.Id
	join T_CFG_SECTION_MASTER SC on SP.Section_Id=SC.id
	where SP.Branch_Id=@pBranchId and SP.YearOfStudy_Id=@pYearOfStudyId and SP.Semester_Id=@pSemesterId and SP.Section_Id =@pSectionId  
	and SP.Delete_Flag=0 and SP.status=1 and SP.TC_ISSUED=0 
	and att.Attendance_Date between @pFromDt and @pToDt
	group by sp.Id,SP.Gender,SP.SNo, SP.StudentId_Rollno, SP.UniversityRegNo,SP.StudentName,SP.Initial,
	SC.Section_Name
	)



	select a.StuTabId,a.StudentName,a.Total,a.rank_No,b.[Attendance Percentage] from TotalInfo a
	left outer join  AttendanceInfo b on a.StudentId_Rollno= b.StudentId_Rollno

	
)

GO
/****** Object:  UserDefinedFunction [dbo].[SplitString]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

create function [dbo].[SplitString] 
    (
        @str nvarchar(4000), 
        @separator char(1)
    )
    returns table
    AS
    return (
        with tokens(p, a, b) AS (
            select 
                1, 
                1, 
                charindex(@separator, @str)
            union all
            select
                p + 1, 
                b + 1, 
                charindex(@separator, @str, b + 1)
            from tokens
            where b > 0
        )
        select
            p-1 zeroBasedOccurance,
            substring(
                @str, 
                a, 
                case when b > 0 then b-a ELSE 4000 end) 
            AS s
        from tokens
      )




GO
/****** Object:  View [dbo].[V_Attendance]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE VIEW [dbo].[V_Attendance]
AS
SELECT        Branch_Id, YearOfStudy_Id, Semester_Id, Section_Id, Course_Id, courseName, StudentName, CAST((PersentPeriod * 100) / (PersentPeriod + AbsentPeriod) 
                         AS decimal(18, 2)) AS PercentageOfAttendance
FROM            (SELECT DISTINCT 
                                                    Branch_Id, YearOfStudy_Id, Semester_Id, Section_Id, Course_Id, courseName, StudentName, 
                                                    SUM(CASE WHEN TempTab.Attendance_Status = 'P' THEN 1 ELSE 0 END) AS PersentPeriod, 
                                                    SUM(CASE WHEN TempTab.Attendance_Status = 'AB' THEN 1 ELSE 0 END) AS AbsentPeriod
                          FROM            (SELECT DISTINCT 
                                                                              AE.Branch_Id, AE.YearOfStudy_Id, AE.Semester_Id, AE.Section_Id, STU.StudentName + ' ' + STU.Initial AS StudentName, 
                                                                              CM.Course_Name AS courseName, AE.Course_Id, AE.Attendance_Status, AE.Attendance_Date, AE.Period
                                                    FROM            dbo.T_ATTENDANCEENTRY AS AE INNER JOIN
                                                                              dbo.T_CFG_STUDENT_PERSONAL_DETAIL AS STU ON AE.StudentTable_Id = STU.Id INNER JOIN
                                                                              dbo.T_CFG_COURSE_MASTER AS CM ON AE.Course_Id = CM.Course_Id
                                                    WHERE        (AE.Attendance_Date BETWEEN '2014-03-10' AND '2014-03-14')) AS TempTab
                          GROUP BY Course_Id, courseName, StudentName, Branch_Id, YearOfStudy_Id, Semester_Id, Section_Id) AS finalTable

GO
/****** Object:  View [dbo].[V_FaillistDashboard]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE VIEW [dbo].[V_FaillistDashboard]
AS
SELECT        S.Institution_Id AS Institution_Id, S.UniversityRegNo, COUNT(S.StudentName) AS FailCount, S.StudentName+' '+S.Initial AS StudentName, b.Branch_Code, 
                         YM.YearOfStudy_Id, Course = STUFF
                             ((SELECT        ',' + CONVERT(varchar(50), CM.Course_ShortCode) AS SumLeaveoddays
                                 FROM            T_CFG_COURSE_MASTER CM JOIN
                                                          T_MARK_ENTRY ME ON CM.Course_Id = ME.Course_Id
                                 WHERE        ME.Marks < 50 AND ME.StudentTable_Id = S.Id FOR XML PATH(''), TYPE ).value('.', 'NVARCHAR(MAX)'), 1, 1, ''), YM.YearOfStudy_Name, 
                         SM.Section_Name, M.Exam_Id
FROM            T_MARK_ENTRY M JOIN
                         T_CFG_STUDENT_PERSONAL_DETAIL S ON M.STUDENTTABLE_ID = S.ID JOIN
                         T_CFG_BRANCH B ON M.BRANCH_ID = B.BRANCH_ID JOIN
                         T_CFG_COURSE_MASTER C ON M.COURSE_ID = C.COURSE_ID JOIN
                         T_CFG_YEAROFSTUDY_MASTER YM ON YM.YearOfStudy_Id = S.YearOfStudy_Id JOIN
                         T_CFG_SECTION_MASTER SM ON SM.ID = S.Section_Id
WHERE        m.Marks < 50 AND M.StudentTable_Id = S.Id AND B.Branch_Id = M.Branch_Id AND 
                         C.Course_Id = M.Course_Id
/* AND S.Delete_Flag=0 AND TC_ISSUED =0 */ GROUP BY S.UniversityRegNo, S.StudentName,S.Initial, B.Branch_Code, S.ID, S.Institution_Id, YM.YearOfStudy_Name, 
                         SM.Section_Name, YM.YearOfStudy_Id, M.Exam_Id

GO
/****** Object:  View [dbo].[v_getallabsentcount]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE VIEW [dbo].[v_getallabsentcount]
AS
SELECT        dbo.T_CFG_STUDENT_PERSONAL_DETAIL.StudentName + ' ' + dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Initial AS Expr1, 
                         dbo.T_CFG_STUDENT_PERSONAL_DETAIL.UniversityRegNo, dbo.T_ATTENDANCEENTRY.Period, dbo.T_ATTENDANCEENTRY.Attendance_Status, 
                         dbo.T_CFG_COURSE_MASTER.Course_Name, dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Id, dbo.T_CFG_COURSE_MASTER.Course_Code, 
                         dbo.T_CFG_COURSE_MASTER.Course_Id, dbo.T_CFG_COURSE_MASTER.No_of_Period
FROM            dbo.T_CFG_STUDENT_PERSONAL_DETAIL INNER JOIN
                         dbo.T_ATTENDANCEENTRY ON dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Id = dbo.T_ATTENDANCEENTRY.StudentTable_Id INNER JOIN
                         dbo.T_CFG_COURSE_MASTER ON dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Branch_Id = dbo.T_CFG_COURSE_MASTER.Course_Id
WHERE        (dbo.T_ATTENDANCEENTRY.Attendance_Status = 'AB')

GO
/****** Object:  View [dbo].[v_getallcount]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE VIEW [dbo].[v_getallcount]
AS
SELECT        dbo.T_CFG_STUDENT_PERSONAL_DETAIL.StudentName + ' ' + dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Initial AS StudentName, 
                         dbo.T_CFG_STUDENT_PERSONAL_DETAIL.UniversityRegNo, dbo.T_ATTENDANCEENTRY.Period, dbo.T_ATTENDANCEENTRY.Attendance_Status, 
                         dbo.T_CFG_COURSE_MASTER.Course_Name, dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Id, dbo.T_CFG_COURSE_MASTER.Course_Code, 
                         dbo.T_CFG_COURSE_MASTER.Course_Id, dbo.T_CFG_COURSE_MASTER.No_of_Period
FROM            dbo.T_CFG_STUDENT_PERSONAL_DETAIL INNER JOIN
                         dbo.T_ATTENDANCEENTRY ON dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Id = dbo.T_ATTENDANCEENTRY.StudentTable_Id INNER JOIN
                         dbo.T_CFG_COURSE_MASTER ON dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Branch_Id = dbo.T_CFG_COURSE_MASTER.Course_Id

GO
/****** Object:  View [dbo].[v_getallpresentcount]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE VIEW [dbo].[v_getallpresentcount]
AS
SELECT        dbo.T_CFG_STUDENT_PERSONAL_DETAIL.StudentName + ' ' + dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Initial AS StudentName, 
                         dbo.T_CFG_STUDENT_PERSONAL_DETAIL.UniversityRegNo, dbo.T_ATTENDANCEENTRY.Period, dbo.T_ATTENDANCEENTRY.Attendance_Status, 
                         dbo.T_CFG_COURSE_MASTER.Course_Name, dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Id, dbo.T_CFG_COURSE_MASTER.Course_Code, 
                         dbo.T_CFG_COURSE_MASTER.Course_Id, dbo.T_CFG_COURSE_MASTER.No_of_Period
FROM            dbo.T_CFG_STUDENT_PERSONAL_DETAIL INNER JOIN
                         dbo.T_ATTENDANCEENTRY ON dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Id = dbo.T_ATTENDANCEENTRY.StudentTable_Id INNER JOIN
                         dbo.T_CFG_COURSE_MASTER ON dbo.T_CFG_STUDENT_PERSONAL_DETAIL.Branch_Id = dbo.T_CFG_COURSE_MASTER.Course_Id
WHERE        (dbo.T_ATTENDANCEENTRY.Attendance_Status = 'P')

GO
/****** Object:  View [dbo].[v_getAUResultWithCollegeName]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO

CREATE VIEW [dbo].[v_getAUResultWithCollegeName]
AS
SELECT        TOP (100) PERCENT CC.CollegeName, GD.RegistrationNumber, GD.CourseCode, GD.Grade, GD.Status
FROM            dbo.T_AURESULTGRADEDETAILS AS GD LEFT OUTER JOIN
                         dbo.t_AUCollegeCode AS CC ON GD.RegistrationNumber LIKE CC.CollegeCode + '%' OR GD.RegistrationNumber BETWEEN CC.Range_From AND 
                         CC.Range_To
ORDER BY CC.CollegeName


GO
/****** Object:  View [dbo].[V_UniversityResults]    Script Date: 10/3/2015 3:19:52 PM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE VIEW [dbo].[V_UniversityResults]
AS
SELECT DISTINCT 
                         TOP (100) PERCENT ins.Institution_Name, sd.StudentName + ' ' + sd.Initial AS StudentName, sd.UniversityRegNo, sd.StudentId_Rollno, b.Branch_Name, 
                         yr.YearOfStudy_Name, sm.Semester_Name, au.Course_Code, au.Grade, au.Internal_Mark, au.External_Mark, au.Result_Status, 
                         au.Result_Published_Year_Id, sd.Id, sd.AcademicBatch_Id, sd.AcademicYear_Id, au.Created_Date
FROM            dbo.T_AUEXAM_RESULTS AS au INNER JOIN
                         dbo.T_CFG_STUDENT_PERSONAL_DETAIL AS sd ON au.Student_Registration_No = sd.UniversityRegNo INNER JOIN
                         dbo.T_CFG_INSTITUTION AS ins ON sd.Institution_Id = ins.Institution_Id INNER JOIN
                         dbo.T_CFG_BRANCH AS b ON sd.Branch_Id = b.Branch_Id INNER JOIN
                         dbo.T_CFG_YEAROFSTUDY_MASTER AS yr ON sd.YearOfStudy_Id = yr.YearOfStudy_Id INNER JOIN
                         dbo.T_CFG_SEMESTER AS sm ON sd.Semester_Id = sm.Semester_Id

GO
ALTER TABLE [dbo].[T_ATTENDANCE_DELETE_HISTORY] ADD  CONSTRAINT [DF_T_ATTENDANCE_DELETE_HISTORY_DELETE_FLAG]  DEFAULT ((0)) FOR [DELETE_FLAG]
GO
ALTER TABLE [dbo].[T_ATTENDANCE_DELETE_HISTORY] ADD  CONSTRAINT [DF_T_ATTENDANCE_DELETE_HISTORY_CREATED_DATE]  DEFAULT (getdate()) FOR [CREATED_DATE]
GO
ALTER TABLE [dbo].[T_ATTENDANCE_HISTRY] ADD  CONSTRAINT [DF_T_ATTENDANCE_HISTRY_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_AU_REVIEW] ADD  CONSTRAINT [DF_tbl_Review_Result_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_AU_REVIEW] ADD  CONSTRAINT [DF_tbl_Review_Result_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_AU_REVIEW] ADD  CONSTRAINT [DF_tbl_Review_Result_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_AUEXAM_RESULTS] ADD  CONSTRAINT [DF_tbl_AUExam_Results_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_AUEXAM_RESULTS] ADD  CONSTRAINT [DF_tbl_AUExam_Results_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_AUEXAM_RESULTS] ADD  CONSTRAINT [DF_tbl_AUExam_Results_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_CFG_ACADEMIC_YEAR] ADD  CONSTRAINT [DF_tbl_cfg_Academic_year_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_ACADEMIC_YEAR] ADD  CONSTRAINT [DF_tbl_cfg_Academic_year_flag]  DEFAULT ((0)) FOR [flag]
GO
ALTER TABLE [dbo].[T_CFG_CONFIGURATON_SETTINGS] ADD  CONSTRAINT [DF_ConfigurationSetting_IsActive]  DEFAULT ((1)) FOR [IsActive]
GO
ALTER TABLE [dbo].[T_CFG_CONFIGURATON_SETTINGS] ADD  CONSTRAINT [DF_ConfigurationSetting_IsDelete]  DEFAULT ((0)) FOR [IsDelete]
GO
ALTER TABLE [dbo].[T_CFG_COURSE_MASTER] ADD  CONSTRAINT [DF_T_CFG_COURSE_MASTER_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_COURSE_MASTER] ADD  CONSTRAINT [DF_T_CFG_COURSE_MASTER_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_COURSE_MASTER] ADD  CONSTRAINT [DF_T_CFG_COURSE_MASTER_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_CFG_COURSE_MASTER] ADD  CONSTRAINT [DF_T_CFG_COURSE_MASTER_Is_Exam_1]  DEFAULT ('1') FOR [Is_Exam]
GO
ALTER TABLE [dbo].[T_CFG_DESIGNATION] ADD  CONSTRAINT [DF_tbl_cfg_Designation_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_DESIGNATION] ADD  CONSTRAINT [DF_tbl_cfg_Designation_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_DESIGNATION] ADD  CONSTRAINT [DF_tbl_cfg_Designation_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_CFG_ELECTIVECOURSE_STAFF_PERIODS] ADD  CONSTRAINT [DF_T_CFG_ELECTIVECOURSE_STAFF_PERIODS_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_ELECTIVECOURSE_STAFF_PERIODS] ADD  CONSTRAINT [DF_T_CFG_ELECTIVECOURSE_STAFF_PERIODS_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_ERRORLOG] ADD  CONSTRAINT [DF_ErrorLog_Data_Date]  DEFAULT (getdate()) FOR [Data_Date]
GO
ALTER TABLE [dbo].[T_CFG_ERRORLOG] ADD  CONSTRAINT [DF_ErrorLog_IsDelete]  DEFAULT ((0)) FOR [IsDelete]
GO
ALTER TABLE [dbo].[T_CFG_INSTITUTION_BRANCH] ADD  CONSTRAINT [DF_tbl_cfg_Institution_Branch_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_INSTITUTION_BRANCH] ADD  CONSTRAINT [DF_tbl_cfg_Institution_Branch_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_LEAVE_REASON] ADD  CONSTRAINT [DF_tbl_cfg_LeaveReason_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_LEAVE_REASON] ADD  CONSTRAINT [DF_tbl_cfg_LeaveReason_IsDeleted]  DEFAULT ((0)) FOR [IsDeleted]
GO
ALTER TABLE [dbo].[T_CFG_LEAVE_REASON] ADD  CONSTRAINT [DF_tbl_cfg_LeaveReason_CreatedBy]  DEFAULT (N'Admin') FOR [CreatedBy]
GO
ALTER TABLE [dbo].[T_CFG_LEAVE_REASON] ADD  CONSTRAINT [DF_tbl_cfg_LeaveReason_CreatedDate]  DEFAULT (getdate()) FOR [CreatedDate]
GO
ALTER TABLE [dbo].[T_CFG_MULTIPLE_STAFF_PERIODS] ADD  CONSTRAINT [DF_T_CFG_MULTIPLE_STAFF_PERIODS_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_MULTIPLE_STAFF_PERIODS] ADD  CONSTRAINT [DF_T_CFG_MULTIPLE_STAFF_PERIODS_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_PERIOD] ADD  CONSTRAINT [DF_tbl_Period_Data_Date]  DEFAULT (getdate()) FOR [Data_Date]
GO
ALTER TABLE [dbo].[T_CFG_PERIOD] ADD  CONSTRAINT [DF_tbl_Period_IsActive]  DEFAULT ((1)) FOR [IsActive]
GO
ALTER TABLE [dbo].[T_CFG_PERIOD] ADD  CONSTRAINT [DF_tbl_Period_Flag]  DEFAULT ((0)) FOR [Flag]
GO
ALTER TABLE [dbo].[T_CFG_Recipients] ADD  CONSTRAINT [DF_T_CFG_Recipients_Active]  DEFAULT ((1)) FOR [Active]
GO
ALTER TABLE [dbo].[T_CFG_Recipients] ADD  CONSTRAINT [DF_T_CFG_Recipients_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_RESULT_PUBLISH_YEAR] ADD  CONSTRAINT [DF_tbl_Result_PublishYear_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_RESULT_PUBLISH_YEAR] ADD  CONSTRAINT [DF_tbl_Result_PublishYear_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_SATURDAY_TIMETABLE] ADD  CONSTRAINT [DF_T_CFG_SATURDAY_TIMETABLE_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_SATURDAY_TIMETABLE] ADD  CONSTRAINT [DF_T_CFG_SATURDAY_TIMETABLE_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_SATURDAY_TIMETABLE] ADD  CONSTRAINT [DF_T_CFG_SATURDAY_TIMETABLE_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_CFG_SEMESTER] ADD  CONSTRAINT [DF_T_CFG_SEMESTER_Active_Semester_1]  DEFAULT ((0)) FOR [Active_Semester]
GO
ALTER TABLE [dbo].[T_CFG_STAFF_LABPERIODS] ADD  CONSTRAINT [DF_T_CFG_STAFF_LABPERIODS_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_STAFF_LABPERIODS] ADD  CONSTRAINT [DF_T_CFG_STAFF_LABPERIODS_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_STAFF_MASTER] ADD  CONSTRAINT [DF_tbl_cfg_Staff_Master_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_STAFF_MASTER] ADD  CONSTRAINT [DF_tbl_cfg_Staff_Master_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_STAFF_MASTER] ADD  CONSTRAINT [DF_tbl_cfg_Staff_Master_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_CFG_STAFF_MASTER] ADD  CONSTRAINT [DF_tbl_cfg_Staff_Master_Modified_Date]  DEFAULT (getdate()) FOR [Modified_Date]
GO
ALTER TABLE [dbo].[T_CFG_STUDENT_PERSONAL_DETAIL] ADD  CONSTRAINT [DF_T_CFG_STUDENT_PERSONAL_DETAIL_Is_Hostel]  DEFAULT ((0)) FOR [Is_Hostel]
GO
ALTER TABLE [dbo].[T_CFG_YEAROFSTUDY_MASTER] ADD  CONSTRAINT [DF_tbl_cfg_YearOfStudy_Master_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_CFG_YEAROFSTUDY_MASTER] ADD  CONSTRAINT [DF_tbl_cfg_YearOfStudy_Master_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_CFG_YEAROFSTUDY_MASTER] ADD  CONSTRAINT [DF_tbl_cfg_YearOfStudy_Master_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_CFG_YEAROFSTUDY_MASTER] ADD  CONSTRAINT [DF_tbl_cfg_YearOfStudy_Master_Modified_Date]  DEFAULT (getdate()) FOR [Modified_Date]
GO
ALTER TABLE [dbo].[T_EDUMATE_REPORTS] ADD  CONSTRAINT [DF_EdumateReports_Active]  DEFAULT ((1)) FOR [Active]
GO
ALTER TABLE [dbo].[T_EDUMATE_REPORTS] ADD  CONSTRAINT [DF_EdumateReports_DeleteFlag]  DEFAULT ((0)) FOR [DeleteFlag]
GO
ALTER TABLE [dbo].[T_EDUMATE_REPORTS] ADD  CONSTRAINT [DF_EdumateReports_DataDate]  DEFAULT (getdate()) FOR [DataDate]
GO
ALTER TABLE [dbo].[T_REVALUATION] ADD  CONSTRAINT [DF_tbl_Revaluation_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_REVALUATION] ADD  CONSTRAINT [DF_tbl_Revaluation_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_REVALUATION] ADD  CONSTRAINT [DF_tbl_Revaluation_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_REVALUATIONCOURSES] ADD  CONSTRAINT [DF_tbl_RevaluationCourses_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_REVALUATIONCOURSES] ADD  CONSTRAINT [DF_tbl_RevaluationCourses_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_REVALUATIONCOURSES] ADD  CONSTRAINT [DF_tbl_RevaluationCourses_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_SMS_CLIENTS] ADD  CONSTRAINT [DF_tbl_clients_Created_By]  DEFAULT (N'SYSADMIN''') FOR [Created_By]
GO
ALTER TABLE [dbo].[T_SMS_CLIENTS] ADD  CONSTRAINT [DF_tbl_clients_Created_Date]  DEFAULT (getdate()) FOR [Created_Date]
GO
ALTER TABLE [dbo].[T_SMS_CLIENTS] ADD  CONSTRAINT [DF_tbl_SMS_Clients_IsActive]  DEFAULT ((1)) FOR [IsActive]
GO
ALTER TABLE [dbo].[T_SMS_CLIENTS] ADD  CONSTRAINT [DF_tbl_SMS_Clients_Flag]  DEFAULT ((0)) FOR [Flag]
GO
ALTER TABLE [dbo].[T_SMS_REPORT_UPLOAD] ADD  CONSTRAINT [DF_tbl_SMSReportupload_datadate_1]  DEFAULT (getdate()) FOR [Data_Date]
GO
ALTER TABLE [dbo].[T_SMS_REPORT_UPLOAD] ADD  CONSTRAINT [DF_tbl_SMSReportupload_IsActive]  DEFAULT ((1)) FOR [IsActive]
GO
ALTER TABLE [dbo].[T_SMS_REPORT_UPLOAD] ADD  CONSTRAINT [DF_tbl_SMSReportupload_Flag]  DEFAULT ((0)) FOR [Flag]
GO
ALTER TABLE [dbo].[T_SMS_RESPONSE] ADD  CONSTRAINT [DF_tbl_SMS_Response_LastUpdated]  DEFAULT (getdate()) FOR [LastUpdated]
GO
ALTER TABLE [dbo].[T_SMS_RESPONSE] ADD  CONSTRAINT [DF_tbl_SMS_Response_datadate_1]  DEFAULT (getdate()) FOR [Data_Date]
GO
ALTER TABLE [dbo].[T_SMS_RESPONSE] ADD  CONSTRAINT [DF_tbl_SMS_Response_IsActive]  DEFAULT ((1)) FOR [IsActive]
GO
ALTER TABLE [dbo].[T_SMS_RESPONSE] ADD  CONSTRAINT [DF_tbl_SMS_Response_Flag]  DEFAULT ((0)) FOR [Flag]
GO
ALTER TABLE [dbo].[T_SMS_STATUS] ADD  CONSTRAINT [DF_tbl_Status_IsActive]  DEFAULT ((1)) FOR [IsActive]
GO
ALTER TABLE [dbo].[T_SMS_STATUS] ADD  CONSTRAINT [DF_tbl_Status_Flag]  DEFAULT ((0)) FOR [Flag]
GO
ALTER TABLE [dbo].[T_STAFF_CATEGORY] ADD  CONSTRAINT [DF_tbl_Staff_Catogary_Status]  DEFAULT ((1)) FOR [Status]
GO
ALTER TABLE [dbo].[T_STAFF_CATEGORY] ADD  CONSTRAINT [DF_tbl_Staff_Catogary_Delete_Flag]  DEFAULT ((0)) FOR [Delete_Flag]
GO
ALTER TABLE [dbo].[T_STAFF_QUALIFICATION] ADD  CONSTRAINT [DF_tbl_Staff_Qualification_To_Month]  DEFAULT ((0)) FOR [To_Month]
GO
ALTER TABLE [dbo].[tbl_Users] ADD  CONSTRAINT [DF_USERS_IS_DELETED]  DEFAULT ((0)) FOR [Is_Deleted]
GO
ALTER TABLE [dbo].[T_ASSIGNING_CLASS_DTL]  WITH CHECK ADD  CONSTRAINT [FK_tbl_Assigning_class_Dtl_tbl_Assigning_class_Hdr] FOREIGN KEY([Assigning_class_ID])
REFERENCES [dbo].[T_ASSIGNING_CLASS_HDR] ([Assigning_class_ID])
GO
ALTER TABLE [dbo].[T_ASSIGNING_CLASS_DTL] CHECK CONSTRAINT [FK_tbl_Assigning_class_Dtl_tbl_Assigning_class_Hdr]
GO
ALTER TABLE [dbo].[T_ATTENDANCEENTRY]  WITH CHECK ADD  CONSTRAINT [FK_tbl_AttendanceEntry_tbl_cfg_Branch] FOREIGN KEY([Branch_Id])
REFERENCES [dbo].[T_CFG_BRANCH] ([Branch_Id])
GO
ALTER TABLE [dbo].[T_ATTENDANCEENTRY] CHECK CONSTRAINT [FK_tbl_AttendanceEntry_tbl_cfg_Branch]
GO
ALTER TABLE [dbo].[T_ATTENDANCEENTRY]  WITH CHECK ADD  CONSTRAINT [FK_tbl_AttendanceEntry_tbl_cfg_Staff_Master] FOREIGN KEY([Staff_Id])
REFERENCES [dbo].[T_CFG_STAFF_MASTER] ([Staff_Id])
GO
ALTER TABLE [dbo].[T_ATTENDANCEENTRY] CHECK CONSTRAINT [FK_tbl_AttendanceEntry_tbl_cfg_Staff_Master]
GO
ALTER TABLE [dbo].[T_ATTENDANCEENTRY_SCHOOL]  WITH CHECK ADD  CONSTRAINT [FK_tbl_AttendanceEntry_School_tbl_cfg_Institution] FOREIGN KEY([Institution_Id])
REFERENCES [dbo].[T_CFG_INSTITUTION] ([Institution_Id])
GO
ALTER TABLE [dbo].[T_ATTENDANCEENTRY_SCHOOL] CHECK CONSTRAINT [FK_tbl_AttendanceEntry_School_tbl_cfg_Institution]
GO
ALTER TABLE [dbo].[T_CFG_BRANCH]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_Branch_tbl_cfg_Department] FOREIGN KEY([Department_Id])
REFERENCES [dbo].[T_CFG_DEPARTMENT] ([Department_Id])
GO
ALTER TABLE [dbo].[T_CFG_BRANCH] CHECK CONSTRAINT [FK_tbl_cfg_Branch_tbl_cfg_Department]
GO
ALTER TABLE [dbo].[T_CFG_BRANCH]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_Branch_tbl_cfg_Programme] FOREIGN KEY([Programme_Id])
REFERENCES [dbo].[T_CFG_PROGRAMME] ([Programme_Id])
GO
ALTER TABLE [dbo].[T_CFG_BRANCH] CHECK CONSTRAINT [FK_tbl_cfg_Branch_tbl_cfg_Programme]
GO
ALTER TABLE [dbo].[T_CFG_DEPARTMENT]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_Department_tbl_cfg_Faculty] FOREIGN KEY([Faculty_Id])
REFERENCES [dbo].[T_CFG_FACULTY] ([Faculty_Id])
GO
ALTER TABLE [dbo].[T_CFG_DEPARTMENT] CHECK CONSTRAINT [FK_tbl_cfg_Department_tbl_cfg_Faculty]
GO
ALTER TABLE [dbo].[T_CFG_INSTITUTION]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_Institution_tbl_cfg_Institution_Group] FOREIGN KEY([Institution_Group_Id])
REFERENCES [dbo].[T_CFG_INSTITUTION_GROUP] ([Institution_Group_Id])
GO
ALTER TABLE [dbo].[T_CFG_INSTITUTION] CHECK CONSTRAINT [FK_tbl_cfg_Institution_tbl_cfg_Institution_Group]
GO
ALTER TABLE [dbo].[T_CFG_INSTITUTION_BRANCH]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_Institution_Branch_tbl_cfg_Branch] FOREIGN KEY([Branch_Id])
REFERENCES [dbo].[T_CFG_BRANCH] ([Branch_Id])
GO
ALTER TABLE [dbo].[T_CFG_INSTITUTION_BRANCH] CHECK CONSTRAINT [FK_tbl_cfg_Institution_Branch_tbl_cfg_Branch]
GO
ALTER TABLE [dbo].[T_CFG_INSTITUTION_BRANCH]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_Institution_Branch_tbl_cfg_Institution] FOREIGN KEY([Institution_Id])
REFERENCES [dbo].[T_CFG_INSTITUTION] ([Institution_Id])
GO
ALTER TABLE [dbo].[T_CFG_INSTITUTION_BRANCH] CHECK CONSTRAINT [FK_tbl_cfg_Institution_Branch_tbl_cfg_Institution]
GO
ALTER TABLE [dbo].[T_CFG_ROLESPERMISSION]  WITH CHECK ADD  CONSTRAINT [FK_RolesPermission_Privilege] FOREIGN KEY([PrivilegeId])
REFERENCES [dbo].[T_CFG_PRIVILEGE] ([Id])
GO
ALTER TABLE [dbo].[T_CFG_ROLESPERMISSION] CHECK CONSTRAINT [FK_RolesPermission_Privilege]
GO
ALTER TABLE [dbo].[T_CFG_STAFF_MASTER]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_Staff_Master_tbl_cfg_Institution] FOREIGN KEY([Institution_Id])
REFERENCES [dbo].[T_CFG_INSTITUTION] ([Institution_Id])
GO
ALTER TABLE [dbo].[T_CFG_STAFF_MASTER] CHECK CONSTRAINT [FK_tbl_cfg_Staff_Master_tbl_cfg_Institution]
GO
ALTER TABLE [dbo].[T_CFG_STATES]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_States_tbl_Country] FOREIGN KEY([Country_Id])
REFERENCES [dbo].[T_CFG_COUNTRY] ([Country_Id])
GO
ALTER TABLE [dbo].[T_CFG_STATES] CHECK CONSTRAINT [FK_tbl_cfg_States_tbl_Country]
GO
ALTER TABLE [dbo].[T_CFG_STUDENT_PERSONAL_DETAIL]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_StudentPersonalDetail_tbl_cfg_Branch] FOREIGN KEY([Branch_Id])
REFERENCES [dbo].[T_CFG_BRANCH] ([Branch_Id])
GO
ALTER TABLE [dbo].[T_CFG_STUDENT_PERSONAL_DETAIL] CHECK CONSTRAINT [FK_tbl_cfg_StudentPersonalDetail_tbl_cfg_Branch]
GO
ALTER TABLE [dbo].[T_CFG_STUDENT_PERSONAL_DETAIL]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_StudentPersonalDetail_tbl_cfg_Institution] FOREIGN KEY([Institution_Id])
REFERENCES [dbo].[T_CFG_INSTITUTION] ([Institution_Id])
GO
ALTER TABLE [dbo].[T_CFG_STUDENT_PERSONAL_DETAIL] CHECK CONSTRAINT [FK_tbl_cfg_StudentPersonalDetail_tbl_cfg_Institution]
GO
ALTER TABLE [dbo].[T_CFG_TIMETABLE_DEFINITION]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_TimeTable_Definition_tbl_cfg_Branch] FOREIGN KEY([Branch_Id])
REFERENCES [dbo].[T_CFG_BRANCH] ([Branch_Id])
GO
ALTER TABLE [dbo].[T_CFG_TIMETABLE_DEFINITION] CHECK CONSTRAINT [FK_tbl_cfg_TimeTable_Definition_tbl_cfg_Branch]
GO
ALTER TABLE [dbo].[T_CFG_TIMETABLE_DEFINITION]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_TimeTable_Definition_tbl_cfg_Institution] FOREIGN KEY([Institution_Id])
REFERENCES [dbo].[T_CFG_INSTITUTION] ([Institution_Id])
GO
ALTER TABLE [dbo].[T_CFG_TIMETABLE_DEFINITION] CHECK CONSTRAINT [FK_tbl_cfg_TimeTable_Definition_tbl_cfg_Institution]
GO
ALTER TABLE [dbo].[T_CFG_TIMETABLE_DEFINITION]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_TimeTable_Definition_tbl_cfg_Institution_Group] FOREIGN KEY([Institution_Group_Id])
REFERENCES [dbo].[T_CFG_INSTITUTION_GROUP] ([Institution_Group_Id])
GO
ALTER TABLE [dbo].[T_CFG_TIMETABLE_DEFINITION] CHECK CONSTRAINT [FK_tbl_cfg_TimeTable_Definition_tbl_cfg_Institution_Group]
GO
ALTER TABLE [dbo].[T_CFG_YEAROFSTUDY_MASTER]  WITH CHECK ADD  CONSTRAINT [FK_tbl_cfg_YearOfStudy_Master_tbl_cfg_Programme] FOREIGN KEY([Programme_Id])
REFERENCES [dbo].[T_CFG_PROGRAMME] ([Programme_Id])
GO
ALTER TABLE [dbo].[T_CFG_YEAROFSTUDY_MASTER] CHECK CONSTRAINT [FK_tbl_cfg_YearOfStudy_Master_tbl_cfg_Programme]
GO
ALTER TABLE [dbo].[T_EXAMDEFINITION_DETAIL]  WITH CHECK ADD  CONSTRAINT [FK_tbl_ExamDefinition_Detail_tbl_Exam_Definition] FOREIGN KEY([Exam_Definition_Id])
REFERENCES [dbo].[T_EXAM_DEFINITION] ([Exam_Definition_Id])
GO
ALTER TABLE [dbo].[T_EXAMDEFINITION_DETAIL] CHECK CONSTRAINT [FK_tbl_ExamDefinition_Detail_tbl_Exam_Definition]
GO
ALTER TABLE [dbo].[T_INPLANT_DETAILS]  WITH CHECK ADD  CONSTRAINT [FK_tbl_Inplant_Details_tbl_Inplant_Hdr] FOREIGN KEY([InplantId])
REFERENCES [dbo].[T_INPLANT_HDR] ([InplantId])
GO
ALTER TABLE [dbo].[T_INPLANT_DETAILS] CHECK CONSTRAINT [FK_tbl_Inplant_Details_tbl_Inplant_Hdr]
GO
ALTER TABLE [dbo].[T_INPLANT_HDR]  WITH CHECK ADD  CONSTRAINT [FK_tbl_Inplant_Hdr_tbl_cfg_Organization] FOREIGN KEY([Organisationid])
REFERENCES [dbo].[T_CFG_ORGANIZATION] ([Org_Id])
GO
ALTER TABLE [dbo].[T_INPLANT_HDR] CHECK CONSTRAINT [FK_tbl_Inplant_Hdr_tbl_cfg_Organization]
GO
ALTER TABLE [dbo].[T_MENTOR_DTL]  WITH CHECK ADD  CONSTRAINT [FK_t_Mentor_dtl_t_Mentor_Hdr] FOREIGN KEY([MENTOR_ID])
REFERENCES [dbo].[T_MENTOR_HDR] ([MENTOR_ID])
GO
ALTER TABLE [dbo].[T_MENTOR_DTL] CHECK CONSTRAINT [FK_t_Mentor_dtl_t_Mentor_Hdr]
GO
ALTER TABLE [dbo].[T_MENTOR_HDR]  WITH CHECK ADD  CONSTRAINT [FK_t_Mentor_Hdr_tbl_cfg_Branch] FOREIGN KEY([BRANCH_ID])
REFERENCES [dbo].[T_CFG_BRANCH] ([Branch_Id])
GO
ALTER TABLE [dbo].[T_MENTOR_HDR] CHECK CONSTRAINT [FK_t_Mentor_Hdr_tbl_cfg_Branch]
GO
ALTER TABLE [dbo].[T_MENTOR_HDR]  WITH CHECK ADD  CONSTRAINT [FK_t_Mentor_Hdr_tbl_cfg_Institution] FOREIGN KEY([INSTITUTION_ID])
REFERENCES [dbo].[T_CFG_INSTITUTION] ([Institution_Id])
GO
ALTER TABLE [dbo].[T_MENTOR_HDR] CHECK CONSTRAINT [FK_t_Mentor_Hdr_tbl_cfg_Institution]
GO
ALTER TABLE [dbo].[T_MENTOR_HDR]  WITH CHECK ADD  CONSTRAINT [FK_t_Mentor_Hdr_tbl_cfg_Staff_Master] FOREIGN KEY([STAFF_ID])
REFERENCES [dbo].[T_CFG_STAFF_MASTER] ([Staff_Id])
GO
ALTER TABLE [dbo].[T_MENTOR_HDR] CHECK CONSTRAINT [FK_t_Mentor_Hdr_tbl_cfg_Staff_Master]
GO
ALTER TABLE [dbo].[T_RETEST_ATTENDANCE_ENTRY]  WITH CHECK ADD  CONSTRAINT [FK_T_RETEST_ATTENDANCE_ENTRY_tbl_cfg_Branch] FOREIGN KEY([Branch_Id])
REFERENCES [dbo].[T_CFG_BRANCH] ([Branch_Id])
GO
ALTER TABLE [dbo].[T_RETEST_ATTENDANCE_ENTRY] CHECK CONSTRAINT [FK_T_RETEST_ATTENDANCE_ENTRY_tbl_cfg_Branch]
GO
ALTER TABLE [dbo].[T_RETEST_ATTENDANCE_ENTRY]  WITH CHECK ADD  CONSTRAINT [FK_T_RETEST_ATTENDANCE_ENTRY_tbl_cfg_Staff_Master] FOREIGN KEY([Staff_Id])
REFERENCES [dbo].[T_CFG_STAFF_MASTER] ([Staff_Id])
GO
ALTER TABLE [dbo].[T_RETEST_ATTENDANCE_ENTRY] CHECK CONSTRAINT [FK_T_RETEST_ATTENDANCE_ENTRY_tbl_cfg_Staff_Master]
GO
ALTER TABLE [dbo].[T_SPONSARSHIP_DTL]  WITH CHECK ADD  CONSTRAINT [FK_tbl_Sponsorship_Dtl_tbl_Sponsorship_Hdr] FOREIGN KEY([Hdr_Id])
REFERENCES [dbo].[T_SPONSARSHIP_HDR] ([Id])
GO
ALTER TABLE [dbo].[T_SPONSARSHIP_DTL] CHECK CONSTRAINT [FK_tbl_Sponsorship_Dtl_tbl_Sponsorship_Hdr]
GO
ALTER TABLE [dbo].[T_SPONSARSHIP_HDR]  WITH CHECK ADD  CONSTRAINT [FK_tbl_Sponsorship_Hdr_tbl_cfg_Branch] FOREIGN KEY([Branch_Id])
REFERENCES [dbo].[T_CFG_BRANCH] ([Branch_Id])
GO
ALTER TABLE [dbo].[T_SPONSARSHIP_HDR] CHECK CONSTRAINT [FK_tbl_Sponsorship_Hdr_tbl_cfg_Branch]
GO
ALTER TABLE [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_DTL]  WITH CHECK ADD  CONSTRAINT [FK_tbl_StaffAssigning_Class_Course_Dtl_tbl_StaffAssigning_Class_Course_Hdr] FOREIGN KEY([Hdr_Id])
REFERENCES [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_HDR] ([Id])
GO
ALTER TABLE [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_DTL] CHECK CONSTRAINT [FK_tbl_StaffAssigning_Class_Course_Dtl_tbl_StaffAssigning_Class_Course_Hdr]
GO
ALTER TABLE [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_HDR]  WITH CHECK ADD  CONSTRAINT [FK_tbl_StaffAssigning_Class_Course_Hdr_tbl_cfg_Branch] FOREIGN KEY([Branch_Id])
REFERENCES [dbo].[T_CFG_BRANCH] ([Branch_Id])
GO
ALTER TABLE [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_HDR] CHECK CONSTRAINT [FK_tbl_StaffAssigning_Class_Course_Hdr_tbl_cfg_Branch]
GO
ALTER TABLE [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_HDR]  WITH CHECK ADD  CONSTRAINT [FK_tbl_StaffAssigning_Class_Course_Hdr_tbl_cfg_Staff_Master] FOREIGN KEY([Staff_Id])
REFERENCES [dbo].[T_CFG_STAFF_MASTER] ([Staff_Id])
GO
ALTER TABLE [dbo].[T_STAFF_ASSIGINING_CLASS_COURSE_HDR] CHECK CONSTRAINT [FK_tbl_StaffAssigning_Class_Course_Hdr_tbl_cfg_Staff_Master]
GO
ALTER TABLE [dbo].[T_STAFF_EXPERIENCE]  WITH CHECK ADD  CONSTRAINT [FK_tbl_Staff_Experience_tbl_cfg_Staff_Master] FOREIGN KEY([Staff_Id])
REFERENCES [dbo].[T_CFG_STAFF_MASTER] ([Staff_Id])
GO
ALTER TABLE [dbo].[T_STAFF_EXPERIENCE] CHECK CONSTRAINT [FK_tbl_Staff_Experience_tbl_cfg_Staff_Master]
GO
ALTER TABLE [dbo].[T_STAFF_QUALIFICATION]  WITH CHECK ADD  CONSTRAINT [FK_tbl_Staff_Qualification_tbl_cfg_Staff_Master] FOREIGN KEY([Staff_Id])
REFERENCES [dbo].[T_CFG_STAFF_MASTER] ([Staff_Id])
GO
ALTER TABLE [dbo].[T_STAFF_QUALIFICATION] CHECK CONSTRAINT [FK_tbl_Staff_Qualification_tbl_cfg_Staff_Master]
GO
ALTER TABLE [dbo].[T_TOUGH_AREA]  WITH CHECK ADD  CONSTRAINT [FK_tbl_ToughArea_tbl_QuestionPaperReview] FOREIGN KEY([questionid])
REFERENCES [dbo].[T_QUESTIONPAPERREVIEW] ([Id])
GO
ALTER TABLE [dbo].[T_TOUGH_AREA] CHECK CONSTRAINT [FK_tbl_ToughArea_tbl_QuestionPaperReview]
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPane1', @value=N'[0E232FF0-B466-11cf-A24F-00AA00A3EFFF, 1.00]
Begin DesignProperties = 
   Begin PaneConfigurations = 
      Begin PaneConfiguration = 0
         NumPanes = 4
         Configuration = "(H (1[41] 4[1] 2[40] 3) )"
      End
      Begin PaneConfiguration = 1
         NumPanes = 3
         Configuration = "(H (1 [50] 4 [25] 3))"
      End
      Begin PaneConfiguration = 2
         NumPanes = 3
         Configuration = "(H (1 [50] 2 [25] 3))"
      End
      Begin PaneConfiguration = 3
         NumPanes = 3
         Configuration = "(H (4 [30] 2 [40] 3))"
      End
      Begin PaneConfiguration = 4
         NumPanes = 2
         Configuration = "(H (1 [56] 3))"
      End
      Begin PaneConfiguration = 5
         NumPanes = 2
         Configuration = "(H (2 [66] 3))"
      End
      Begin PaneConfiguration = 6
         NumPanes = 2
         Configuration = "(H (4 [50] 3))"
      End
      Begin PaneConfiguration = 7
         NumPanes = 1
         Configuration = "(V (3))"
      End
      Begin PaneConfiguration = 8
         NumPanes = 3
         Configuration = "(H (1[56] 4[18] 2) )"
      End
      Begin PaneConfiguration = 9
         NumPanes = 2
         Configuration = "(H (1 [75] 4))"
      End
      Begin PaneConfiguration = 10
         NumPanes = 2
         Configuration = "(H (1[66] 2) )"
      End
      Begin PaneConfiguration = 11
         NumPanes = 2
         Configuration = "(H (4 [60] 2))"
      End
      Begin PaneConfiguration = 12
         NumPanes = 1
         Configuration = "(H (1) )"
      End
      Begin PaneConfiguration = 13
         NumPanes = 1
         Configuration = "(V (4))"
      End
      Begin PaneConfiguration = 14
         NumPanes = 1
         Configuration = "(V (2))"
      End
      ActivePaneConfig = 0
   End
   Begin DiagramPane = 
      Begin Origin = 
         Top = -96
         Left = 0
      End
      Begin Tables = 
         Begin Table = "finalTable"
            Begin Extent = 
               Top = 102
               Left = 38
               Bottom = 232
               Right = 208
            End
            DisplayFlags = 280
            TopColumn = 0
         End
      End
   End
   Begin SQLPane = 
   End
   Begin DataPane = 
      Begin ParameterDefaults = ""
      End
      Begin ColumnWidths = 9
         Width = 284
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
      End
   End
   Begin CriteriaPane = 
      Begin ColumnWidths = 11
         Column = 7935
         Alias = 2145
         Table = 1170
         Output = 720
         Append = 1400
         NewValue = 1170
         SortType = 1350
         SortOrder = 1410
         GroupBy = 1350
         Filter = 1350
         Or = 1350
         Or = 1350
         Or = 1350
      End
   End
End
' , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'V_Attendance'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPaneCount', @value=1 , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'V_Attendance'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPane1', @value=N'[0E232FF0-B466-11cf-A24F-00AA00A3EFFF, 1.00]
Begin DesignProperties = 
   Begin PaneConfigurations = 
      Begin PaneConfiguration = 0
         NumPanes = 4
         Configuration = "(H (1[13] 4[4] 2[41] 3) )"
      End
      Begin PaneConfiguration = 1
         NumPanes = 3
         Configuration = "(H (1 [50] 4 [25] 3))"
      End
      Begin PaneConfiguration = 2
         NumPanes = 3
         Configuration = "(H (1 [50] 2 [25] 3))"
      End
      Begin PaneConfiguration = 3
         NumPanes = 3
         Configuration = "(H (4 [30] 2 [40] 3))"
      End
      Begin PaneConfiguration = 4
         NumPanes = 2
         Configuration = "(H (1 [56] 3))"
      End
      Begin PaneConfiguration = 5
         NumPanes = 2
         Configuration = "(H (2 [66] 3))"
      End
      Begin PaneConfiguration = 6
         NumPanes = 2
         Configuration = "(H (4 [50] 3))"
      End
      Begin PaneConfiguration = 7
         NumPanes = 1
         Configuration = "(V (3))"
      End
      Begin PaneConfiguration = 8
         NumPanes = 3
         Configuration = "(H (1[56] 4[18] 2) )"
      End
      Begin PaneConfiguration = 9
         NumPanes = 2
         Configuration = "(H (1 [75] 4))"
      End
      Begin PaneConfiguration = 10
         NumPanes = 2
         Configuration = "(H (1[66] 2) )"
      End
      Begin PaneConfiguration = 11
         NumPanes = 2
         Configuration = "(H (4 [60] 2))"
      End
      Begin PaneConfiguration = 12
         NumPanes = 1
         Configuration = "(H (1) )"
      End
      Begin PaneConfiguration = 13
         NumPanes = 1
         Configuration = "(V (4))"
      End
      Begin PaneConfiguration = 14
         NumPanes = 1
         Configuration = "(V (2))"
      End
      ActivePaneConfig = 0
   End
   Begin DiagramPane = 
      Begin Origin = 
         Top = 0
         Left = 0
      End
      Begin Tables = 
      End
   End
   Begin SQLPane = 
   End
   Begin DataPane = 
      Begin ParameterDefaults = ""
      End
      Begin ColumnWidths = 11
         Width = 284
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
      End
   End
   Begin CriteriaPane = 
      Begin ColumnWidths = 11
         Column = 1440
         Alias = 900
         Table = 1170
         Output = 720
         Append = 1400
         NewValue = 1170
         SortType = 1350
         SortOrder = 1410
         GroupBy = 1350
         Filter = 1350
         Or = 1350
         Or = 1350
         Or = 1350
      End
   End
End
' , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'V_FaillistDashboard'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPaneCount', @value=1 , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'V_FaillistDashboard'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPane1', @value=N'[0E232FF0-B466-11cf-A24F-00AA00A3EFFF, 1.00]
Begin DesignProperties = 
   Begin PaneConfigurations = 
      Begin PaneConfiguration = 0
         NumPanes = 4
         Configuration = "(H (1[40] 4[20] 2[20] 3) )"
      End
      Begin PaneConfiguration = 1
         NumPanes = 3
         Configuration = "(H (1 [50] 4 [25] 3))"
      End
      Begin PaneConfiguration = 2
         NumPanes = 3
         Configuration = "(H (1 [50] 2 [25] 3))"
      End
      Begin PaneConfiguration = 3
         NumPanes = 3
         Configuration = "(H (4 [30] 2 [40] 3))"
      End
      Begin PaneConfiguration = 4
         NumPanes = 2
         Configuration = "(H (1 [56] 3))"
      End
      Begin PaneConfiguration = 5
         NumPanes = 2
         Configuration = "(H (2 [66] 3))"
      End
      Begin PaneConfiguration = 6
         NumPanes = 2
         Configuration = "(H (4 [50] 3))"
      End
      Begin PaneConfiguration = 7
         NumPanes = 1
         Configuration = "(V (3))"
      End
      Begin PaneConfiguration = 8
         NumPanes = 3
         Configuration = "(H (1[56] 4[18] 2) )"
      End
      Begin PaneConfiguration = 9
         NumPanes = 2
         Configuration = "(H (1 [75] 4))"
      End
      Begin PaneConfiguration = 10
         NumPanes = 2
         Configuration = "(H (1[66] 2) )"
      End
      Begin PaneConfiguration = 11
         NumPanes = 2
         Configuration = "(H (4 [60] 2))"
      End
      Begin PaneConfiguration = 12
         NumPanes = 1
         Configuration = "(H (1) )"
      End
      Begin PaneConfiguration = 13
         NumPanes = 1
         Configuration = "(V (4))"
      End
      Begin PaneConfiguration = 14
         NumPanes = 1
         Configuration = "(V (2))"
      End
      ActivePaneConfig = 0
   End
   Begin DiagramPane = 
      Begin Origin = 
         Top = 0
         Left = 0
      End
      Begin Tables = 
         Begin Table = "T_CFG_STUDENT_PERSONAL_DETAIL"
            Begin Extent = 
               Top = 6
               Left = 38
               Bottom = 136
               Right = 257
            End
            DisplayFlags = 280
            TopColumn = 0
         End
         Begin Table = "T_ATTENDANCEENTRY"
            Begin Extent = 
               Top = 138
               Left = 38
               Bottom = 268
               Right = 253
            End
            DisplayFlags = 280
            TopColumn = 0
         End
         Begin Table = "T_CFG_COURSE_MASTER"
            Begin Extent = 
               Top = 270
               Left = 38
               Bottom = 400
               Right = 225
            End
            DisplayFlags = 280
            TopColumn = 0
         End
      End
   End
   Begin SQLPane = 
   End
   Begin DataPane = 
      Begin ParameterDefaults = ""
      End
   End
   Begin CriteriaPane = 
      Begin ColumnWidths = 11
         Column = 1440
         Alias = 900
         Table = 1170
         Output = 720
         Append = 1400
         NewValue = 1170
         SortType = 1350
         SortOrder = 1410
         GroupBy = 1350
         Filter = 1350
         Or = 1350
         Or = 1350
         Or = 1350
      End
   End
End
' , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'v_getallabsentcount'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPaneCount', @value=1 , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'v_getallabsentcount'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPane1', @value=N'[0E232FF0-B466-11cf-A24F-00AA00A3EFFF, 1.00]
Begin DesignProperties = 
   Begin PaneConfigurations = 
      Begin PaneConfiguration = 0
         NumPanes = 4
         Configuration = "(H (1[40] 4[20] 2[20] 3) )"
      End
      Begin PaneConfiguration = 1
         NumPanes = 3
         Configuration = "(H (1 [50] 4 [25] 3))"
      End
      Begin PaneConfiguration = 2
         NumPanes = 3
         Configuration = "(H (1 [50] 2 [25] 3))"
      End
      Begin PaneConfiguration = 3
         NumPanes = 3
         Configuration = "(H (4 [30] 2 [40] 3))"
      End
      Begin PaneConfiguration = 4
         NumPanes = 2
         Configuration = "(H (1 [56] 3))"
      End
      Begin PaneConfiguration = 5
         NumPanes = 2
         Configuration = "(H (2 [66] 3))"
      End
      Begin PaneConfiguration = 6
         NumPanes = 2
         Configuration = "(H (4 [50] 3))"
      End
      Begin PaneConfiguration = 7
         NumPanes = 1
         Configuration = "(V (3))"
      End
      Begin PaneConfiguration = 8
         NumPanes = 3
         Configuration = "(H (1[56] 4[18] 2) )"
      End
      Begin PaneConfiguration = 9
         NumPanes = 2
         Configuration = "(H (1 [75] 4))"
      End
      Begin PaneConfiguration = 10
         NumPanes = 2
         Configuration = "(H (1[66] 2) )"
      End
      Begin PaneConfiguration = 11
         NumPanes = 2
         Configuration = "(H (4 [60] 2))"
      End
      Begin PaneConfiguration = 12
         NumPanes = 1
         Configuration = "(H (1) )"
      End
      Begin PaneConfiguration = 13
         NumPanes = 1
         Configuration = "(V (4))"
      End
      Begin PaneConfiguration = 14
         NumPanes = 1
         Configuration = "(V (2))"
      End
      ActivePaneConfig = 0
   End
   Begin DiagramPane = 
      Begin Origin = 
         Top = 0
         Left = 0
      End
      Begin Tables = 
         Begin Table = "T_CFG_STUDENT_PERSONAL_DETAIL"
            Begin Extent = 
               Top = 6
               Left = 38
               Bottom = 136
               Right = 257
            End
            DisplayFlags = 280
            TopColumn = 0
         End
         Begin Table = "T_ATTENDANCEENTRY"
            Begin Extent = 
               Top = 138
               Left = 38
               Bottom = 268
               Right = 253
            End
            DisplayFlags = 280
            TopColumn = 0
         End
         Begin Table = "T_CFG_COURSE_MASTER"
            Begin Extent = 
               Top = 270
               Left = 38
               Bottom = 400
               Right = 225
            End
            DisplayFlags = 280
            TopColumn = 0
         End
      End
   End
   Begin SQLPane = 
   End
   Begin DataPane = 
      Begin ParameterDefaults = ""
      End
   End
   Begin CriteriaPane = 
      Begin ColumnWidths = 11
         Column = 1440
         Alias = 900
         Table = 1170
         Output = 720
         Append = 1400
         NewValue = 1170
         SortType = 1350
         SortOrder = 1410
         GroupBy = 1350
         Filter = 1350
         Or = 1350
         Or = 1350
         Or = 1350
      End
   End
End
' , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'v_getallcount'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPaneCount', @value=1 , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'v_getallcount'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPane1', @value=N'[0E232FF0-B466-11cf-A24F-00AA00A3EFFF, 1.00]
Begin DesignProperties = 
   Begin PaneConfigurations = 
      Begin PaneConfiguration = 0
         NumPanes = 4
         Configuration = "(H (1[40] 4[20] 2[20] 3) )"
      End
      Begin PaneConfiguration = 1
         NumPanes = 3
         Configuration = "(H (1 [50] 4 [25] 3))"
      End
      Begin PaneConfiguration = 2
         NumPanes = 3
         Configuration = "(H (1 [50] 2 [25] 3))"
      End
      Begin PaneConfiguration = 3
         NumPanes = 3
         Configuration = "(H (4 [30] 2 [40] 3))"
      End
      Begin PaneConfiguration = 4
         NumPanes = 2
         Configuration = "(H (1 [56] 3))"
      End
      Begin PaneConfiguration = 5
         NumPanes = 2
         Configuration = "(H (2 [66] 3))"
      End
      Begin PaneConfiguration = 6
         NumPanes = 2
         Configuration = "(H (4 [50] 3))"
      End
      Begin PaneConfiguration = 7
         NumPanes = 1
         Configuration = "(V (3))"
      End
      Begin PaneConfiguration = 8
         NumPanes = 3
         Configuration = "(H (1[56] 4[18] 2) )"
      End
      Begin PaneConfiguration = 9
         NumPanes = 2
         Configuration = "(H (1 [75] 4))"
      End
      Begin PaneConfiguration = 10
         NumPanes = 2
         Configuration = "(H (1[66] 2) )"
      End
      Begin PaneConfiguration = 11
         NumPanes = 2
         Configuration = "(H (4 [60] 2))"
      End
      Begin PaneConfiguration = 12
         NumPanes = 1
         Configuration = "(H (1) )"
      End
      Begin PaneConfiguration = 13
         NumPanes = 1
         Configuration = "(V (4))"
      End
      Begin PaneConfiguration = 14
         NumPanes = 1
         Configuration = "(V (2))"
      End
      ActivePaneConfig = 0
   End
   Begin DiagramPane = 
      Begin Origin = 
         Top = 0
         Left = 0
      End
      Begin Tables = 
         Begin Table = "T_CFG_STUDENT_PERSONAL_DETAIL"
            Begin Extent = 
               Top = 6
               Left = 38
               Bottom = 136
               Right = 257
            End
            DisplayFlags = 280
            TopColumn = 0
         End
         Begin Table = "T_ATTENDANCEENTRY"
            Begin Extent = 
               Top = 138
               Left = 38
               Bottom = 268
               Right = 253
            End
            DisplayFlags = 280
            TopColumn = 0
         End
         Begin Table = "T_CFG_COURSE_MASTER"
            Begin Extent = 
               Top = 270
               Left = 38
               Bottom = 400
               Right = 225
            End
            DisplayFlags = 280
            TopColumn = 0
         End
      End
   End
   Begin SQLPane = 
   End
   Begin DataPane = 
      Begin ParameterDefaults = ""
      End
   End
   Begin CriteriaPane = 
      Begin ColumnWidths = 11
         Column = 1440
         Alias = 900
         Table = 1170
         Output = 720
         Append = 1400
         NewValue = 1170
         SortType = 1350
         SortOrder = 1410
         GroupBy = 1350
         Filter = 1350
         Or = 1350
         Or = 1350
         Or = 1350
      End
   End
End
' , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'v_getallpresentcount'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPaneCount', @value=1 , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'v_getallpresentcount'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPane1', @value=N'[0E232FF0-B466-11cf-A24F-00AA00A3EFFF, 1.00]
Begin DesignProperties = 
   Begin PaneConfigurations = 
      Begin PaneConfiguration = 0
         NumPanes = 4
         Configuration = "(H (1[41] 4[20] 2[3] 3) )"
      End
      Begin PaneConfiguration = 1
         NumPanes = 3
         Configuration = "(H (1 [50] 4 [25] 3))"
      End
      Begin PaneConfiguration = 2
         NumPanes = 3
         Configuration = "(H (1 [50] 2 [25] 3))"
      End
      Begin PaneConfiguration = 3
         NumPanes = 3
         Configuration = "(H (4 [30] 2 [40] 3))"
      End
      Begin PaneConfiguration = 4
         NumPanes = 2
         Configuration = "(H (1 [56] 3))"
      End
      Begin PaneConfiguration = 5
         NumPanes = 2
         Configuration = "(H (2 [66] 3))"
      End
      Begin PaneConfiguration = 6
         NumPanes = 2
         Configuration = "(H (4 [50] 3))"
      End
      Begin PaneConfiguration = 7
         NumPanes = 1
         Configuration = "(V (3))"
      End
      Begin PaneConfiguration = 8
         NumPanes = 3
         Configuration = "(H (1[56] 4[18] 2) )"
      End
      Begin PaneConfiguration = 9
         NumPanes = 2
         Configuration = "(H (1 [75] 4))"
      End
      Begin PaneConfiguration = 10
         NumPanes = 2
         Configuration = "(H (1[66] 2) )"
      End
      Begin PaneConfiguration = 11
         NumPanes = 2
         Configuration = "(H (4 [60] 2))"
      End
      Begin PaneConfiguration = 12
         NumPanes = 1
         Configuration = "(H (1) )"
      End
      Begin PaneConfiguration = 13
         NumPanes = 1
         Configuration = "(V (4))"
      End
      Begin PaneConfiguration = 14
         NumPanes = 1
         Configuration = "(V (2))"
      End
      ActivePaneConfig = 0
   End
   Begin DiagramPane = 
      Begin Origin = 
         Top = 0
         Left = 0
      End
      Begin Tables = 
         Begin Table = "CC"
            Begin Extent = 
               Top = 6
               Left = 272
               Bottom = 136
               Right = 442
            End
            DisplayFlags = 280
            TopColumn = 0
         End
         Begin Table = "GD"
            Begin Extent = 
               Top = 6
               Left = 38
               Bottom = 136
               Right = 234
            End
            DisplayFlags = 280
            TopColumn = 0
         End
      End
   End
   Begin SQLPane = 
   End
   Begin DataPane = 
      Begin ParameterDefaults = ""
      End
      Begin ColumnWidths = 9
         Width = 284
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
      End
   End
   Begin CriteriaPane = 
      Begin ColumnWidths = 11
         Column = 1440
         Alias = 900
         Table = 1170
         Output = 720
         Append = 1400
         NewValue = 1170
         SortType = 1350
         SortOrder = 1410
         GroupBy = 1350
         Filter = 1350
         Or = 1350
         Or = 1350
         Or = 1350
      End
   End
End
' , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'v_getAUResultWithCollegeName'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPaneCount', @value=1 , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'v_getAUResultWithCollegeName'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPane1', @value=N'[0E232FF0-B466-11cf-A24F-00AA00A3EFFF, 1.00]
Begin DesignProperties = 
   Begin PaneConfigurations = 
      Begin PaneConfiguration = 0
         NumPanes = 4
         Configuration = "(H (1[20] 4[29] 2[26] 3) )"
      End
      Begin PaneConfiguration = 1
         NumPanes = 3
         Configuration = "(H (1 [50] 4 [25] 3))"
      End
      Begin PaneConfiguration = 2
         NumPanes = 3
         Configuration = "(H (1 [50] 2 [25] 3))"
      End
      Begin PaneConfiguration = 3
         NumPanes = 3
         Configuration = "(H (4 [30] 2 [40] 3))"
      End
      Begin PaneConfiguration = 4
         NumPanes = 2
         Configuration = "(H (1 [56] 3))"
      End
      Begin PaneConfiguration = 5
         NumPanes = 2
         Configuration = "(H (2 [66] 3))"
      End
      Begin PaneConfiguration = 6
         NumPanes = 2
         Configuration = "(H (4 [50] 3))"
      End
      Begin PaneConfiguration = 7
         NumPanes = 1
         Configuration = "(V (3))"
      End
      Begin PaneConfiguration = 8
         NumPanes = 3
         Configuration = "(H (1[56] 4[18] 2) )"
      End
      Begin PaneConfiguration = 9
         NumPanes = 2
         Configuration = "(H (1 [75] 4))"
      End
      Begin PaneConfiguration = 10
         NumPanes = 2
         Configuration = "(H (1[66] 2) )"
      End
      Begin PaneConfiguration = 11
         NumPanes = 2
         Configuration = "(H (4 [60] 2))"
      End
      Begin PaneConfiguration = 12
         NumPanes = 1
         Configuration = "(H (1) )"
      End
      Begin PaneConfiguration = 13
         NumPanes = 1
         Configuration = "(V (4))"
      End
      Begin PaneConfiguration = 14
         NumPanes = 1
         Configuration = "(V (2))"
      End
      ActivePaneConfig = 0
   End
   Begin DiagramPane = 
      Begin Origin = 
         Top = -384
         Left = 0
      End
      Begin Tables = 
         Begin Table = "au"
            Begin Extent = 
               Top = 6
               Left = 38
               Bottom = 136
               Right = 259
            End
            DisplayFlags = 280
            TopColumn = 12
         End
         Begin Table = "sd"
            Begin Extent = 
               Top = 138
               Left = 38
               Bottom = 268
               Right = 256
            End
            DisplayFlags = 280
            TopColumn = 11
         End
         Begin Table = "ins"
            Begin Extent = 
               Top = 270
               Left = 38
               Bottom = 400
               Right = 234
            End
            DisplayFlags = 280
            TopColumn = 19
         End
         Begin Table = "b"
            Begin Extent = 
               Top = 270
               Left = 272
               Bottom = 400
               Right = 459
            End
            DisplayFlags = 280
            TopColumn = 10
         End
         Begin Table = "yr"
            Begin Extent = 
               Top = 402
               Left = 38
               Bottom = 532
               Right = 230
            End
            DisplayFlags = 280
            TopColumn = 0
         End
         Begin Table = "sm"
            Begin Extent = 
               Top = 6
               Left = 297
               Bottom = 136
               Right = 471
            End
            DisplayFlags = 280
            TopColumn = 0
         End
      End
   End
   Begin SQLPane = 
   End
   Begin DataPane = 
      Begin ParameterDefaults = ""
      End
      Begin ColumnWidths = 22
         Width = 284
         Width = 1500
         Width = 1500
         Width = 1500
      ' , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'V_UniversityResults'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPane2', @value=N'   Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
         Width = 1500
      End
   End
   Begin CriteriaPane = 
      Begin ColumnWidths = 11
         Column = 1440
         Alias = 900
         Table = 1170
         Output = 720
         Append = 1400
         NewValue = 1170
         SortType = 1350
         SortOrder = 1410
         GroupBy = 1350
         Filter = 1350
         Or = 2130
         Or = 1350
         Or = 1350
      End
   End
End
' , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'V_UniversityResults'
GO
EXEC sys.sp_addextendedproperty @name=N'MS_DiagramPaneCount', @value=2 , @level0type=N'SCHEMA',@level0name=N'dbo', @level1type=N'VIEW',@level1name=N'V_UniversityResults'
GO
