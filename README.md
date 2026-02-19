# Epicor-Production-Efficiency-Report
Production Efficiency Report
/*  
 *BAQ gives you the concept of how to create the production efficiency report and then make it a dashboard in App Studio for different users to get more visibility on the production or shop floor side of the business. 
 */

select  
	[SQ_FinalOper].[JobOper_OpCode] as [JobOper_OpCode], 
	[SQ_ActualOutput].[PartTran_JobNum] as [PartTran_JobNum], 
	[SQ_ActualOutput].[JobHead_ReqDueDate] as [JobHead_ReqDueDate], 
	[SQ_ActualOutput].[PartTran_PartNum] as [PartTran_PartNum], 
	[SQ_FinalOper].[Calculated_StandardOutput] as [Calculated_StandardOutput], 
	[SQ_ActualOutput].[Calculated_ActualOutputQty] as [Calculated_ActualOutputQty], 
	[SQ_ActualOutput].[PartTran_UM] as [PartTran_UM], 
	(SQ_ActualOutput.Calculated_ActualOutputQty-SQ_FinalOper.Calculated_StandardOutput) as [Calculated_Output_Variance], 
	[SQ_FinalOper].[Calculated_StandardOutputKG] as [Calculated_StandardOutputKG], 
	[SQ_ActualOutput].[Calculated_ActualOutputKG] as [Calculated_ActualOutputKG], 
	(SQ_ActualOutput.Calculated_ActualOutputKG -SQ_FinalOper.Calculated_StandardOutputKG) as [Calculated_Output_Variance_KG], 
	(CASE
  WHEN COALESCE(Calculated_StandardOutputKG,0) = 0 THEN 0
  WHEN (Calculated_ActualOutputKG / Calculated_StandardOutputKG) * 100 > 100 THEN 100
  ELSE (Calculated_ActualOutputKG / Calculated_StandardOutputKG) * 100
END) as [Calculated_Pct_Actual_vs_StandardKG] 

from  (select  
	[LaborDtl].[Company] as [LaborDtl_Company], 
	[LaborDtl].[JobNum] as [LaborDtl_JobNum], 
	[LaborDtl].[OprSeq] as [LaborDtl_OprSeq], 
	(sum(LaborDtl.LaborHrs)) as [Calculated_ActualHours], 
	[LaborDtl].[OpCode] as [LaborDtl_OpCode] 

from Erp.LaborDtl as [LaborDtl]
where (LaborDtl.JobNum <> '')
group by 
	[LaborDtl].[Company], 
	[LaborDtl].[JobNum], 
	[LaborDtl].[OprSeq], 
	[LaborDtl].[OpCode])  as [SQ_ActualHours]
right outer join  (select  
	[JobOper].[Company] as [JobOper_Company], 
	[JobOper].[OpCode] as [JobOper_OpCode], 
	[JobOper].[JobNum] as [JobOper_JobNum], 
	(JobOper.ProdStandard*JobOper.EstProdHours) as [Calculated_StandardOutput], 
	[JobOper].[IUM] as [JobOper_IUM], 
	(JobOper.EstProdHours *JobOper.ProdCrewSize) as [Calculated_Calc_EstimatedHours], 
	(StandardOutput  * MAX(COALESCE(Part1.NetWeight, 0))) as [Calculated_StandardOutputKG], 
	[Part1].[NetWeight] as [Part1_NetWeight] 

from Erp.JobOper as [JobOper]
inner join Erp.JobOpDtl as [JobOpDtl] on 
	  JobOper.Company = JobOpDtl.Company
	and  JobOper.JobNum = JobOpDtl.JobNum
	and  JobOper.AssemblySeq = JobOpDtl.AssemblySeq
	and  JobOper.OprSeq = JobOpDtl.OprSeq
left outer join Erp.JobHead as [JobHead2] on 
	  JobOper.Company = JobHead2.Company
	and  JobOper.JobNum = JobHead2.JobNum
left outer join Erp.Part as [Part1] on 
	  JobHead2.Company = Part1.Company
	and  JobHead2.PartNum = Part1.PartNum
where (JobOper.OpCode IN ('Baggar', 'Cartoner', 'Handpack', 'HiCareO', 'VacPack'))
group by 
	[JobOper].[Company], 
	[JobOper].[OpCode], 
	[JobOper].[JobNum], 
	(JobOper.ProdStandard*JobOper.EstProdHours), 
	[JobOper].[IUM], 
	(JobOper.EstProdHours *JobOper.ProdCrewSize), 
	[Part1].[NetWeight])  as [SQ_FinalOper] on 
	  SQ_FinalOper.JobOper_Company = SQ_ActualHours.LaborDtl_Company
	and  SQ_FinalOper.JobOper_JobNum = SQ_ActualHours.LaborDtl_JobNum
	and  SQ_FinalOper.JobOper_OpCode = SQ_ActualHours.LaborDtl_OpCode
right outer join  (select  
	[PartTran].[Company] as [PartTran_Company], 
	[PartTran].[JobNum] as [PartTran_JobNum], 
	[JobHead].[ReqDueDate] as [JobHead_ReqDueDate], 
	[PartTran].[PartNum] as [PartTran_PartNum], 
	[PartTran].[UM] as [PartTran_UM], 
	(sum(PartTran.TranQty * Part.NetWeight)) as [Calculated_ActualOutputKG], 
	(sum(PartTran.TranQty)) as [Calculated_ActualOutputQty], 
	(SUM(PartTran.TranQty * COALESCE(Part.NetWeight,0))) as [Calculated_Est_StandardOutputKG] 

from Erp.PartTran as [PartTran]
left outer join Erp.Part as [Part] on 
	  PartTran.Company = Part.Company
	and  PartTran.PartNum = Part.PartNum
left outer join Erp.JobHead as [JobHead] on 
	  PartTran.Company = JobHead.Company
	and  PartTran.JobNum = JobHead.JobNum
where (PartTran.TranType = 'MFG-STK')
group by 
	[PartTran].[Company], 
	[PartTran].[JobNum], 
	[JobHead].[ReqDueDate], 
	[PartTran].[PartNum], 
	[PartTran].[UM])  as [SQ_ActualOutput] on 
	  SQ_ActualOutput.PartTran_Company = SQ_FinalOper.JobOper_Company
	and  SQ_ActualOutput.PartTran_JobNum = SQ_FinalOper.JobOper_JobNum
inner join Erp.JobHead as [JobHead1] on 
	  SQ_FinalOper.JobOper_Company = JobHead1.Company
	and  SQ_FinalOper.JobOper_JobNum = JobHead1.JobNum
group by 
	[SQ_FinalOper].[JobOper_OpCode], 
	[SQ_ActualOutput].[PartTran_JobNum], 
	[SQ_ActualOutput].[JobHead_ReqDueDate], 
	[SQ_ActualOutput].[PartTran_PartNum], 
	[SQ_FinalOper].[Calculated_StandardOutput], 
	[SQ_ActualOutput].[Calculated_ActualOutputQty], 
	[SQ_ActualOutput].[PartTran_UM], 
	(SQ_ActualOutput.Calculated_ActualOutputQty-SQ_FinalOper.Calculated_StandardOutput), 
	[SQ_FinalOper].[Calculated_StandardOutputKG], 
	[SQ_ActualOutput].[Calculated_ActualOutputKG], 
	(SQ_ActualOutput.Calculated_ActualOutputKG -SQ_FinalOper.Calculated_StandardOutputKG), 
	(CASE
  WHEN COALESCE(Calculated_StandardOutputKG,0) = 0 THEN 0
  WHEN (Calculated_ActualOutputKG / Calculated_StandardOutputKG) * 100 > 100 THEN 100
  ELSE (Calculated_ActualOutputKG / Calculated_StandardOutputKG) * 100
END)
