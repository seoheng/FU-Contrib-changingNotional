	USE [MOS_Reporting]
	GO
	/****** Object:  StoredProcedure [dbo].[PerfRpt_FUDailyContrib_2016_MAS]    Script Date: 2/27/2017 9:42:15 AM ******/
	SET ANSI_NULLS ON
	GO
	SET QUOTED_IDENTIFIER ON
	GO

	-- =======================================================================================================================================
	-- Author:		Lau Seo Heng
	-- Create date: 26-Sep-16
	-- Description:	1. This stored procedure is called from MOS Reports Manager
	--              2. The purpose of the stored procedure is to retrieve and compute returns for futures
	--              3. The CRM Relative Portfolios NAV is bgn of day NAV
	-- Change date: 2016-09-26 - (C005) To take into account of daily changing notional values    
	-- =======================================================================================================================================


	ALTER procedure [dbo].[PerfRpt_FUDailyContrib_2016_MAS] --to take into account of daily changing notional values


		-- Add the parameters for the stored procedure here
		@RevalDateStart		datetime,
		@RevalDateEnd		datetime,
		@Port               varchar(100)

	as
	begin

	
		DECLARE @RunTypeId			int
		DECLARE @portcode			varchar(100)
		DECLARE @Portfolio			varchar(100)
		DECLARE @RevalDate          Datetime


		Set @RunTypeId = 1
		Set @portcode = 'portcode_A,portcode_B'
		--SET @RevalDate = @RevalDateStart
		SET @RevalDate = '2016-10-01'

		if OBJECT_ID('tempdb..#TempFURtnsSummary') IS NOT NULL 	DROP TABLE  #TempFURtnsSummary;
		if OBJECT_ID('tempdb..#TempFUContribSummary') IS NOT NULL 	DROP TABLE  #TempFUContribSummary;
		if OBJECT_ID('tempdb..#TempFUNotionalStaging0') IS NOT NULL 	DROP TABLE  #TempFUNotionalStaging0;
		if OBJECT_ID('tempdb..#TempFUNotionalStaging1') IS NOT NULL 	DROP TABLE  #TempFUNotionalStaging1;
		if OBJECT_ID('tempdb..#TempFUNotionalStaging2') IS NOT NULL 	DROP TABLE  #TempFUNotionalStaging2;
	
		create table #TempFURtnsSummary(
		[reval_date] [datetime],
		[asset_class] [varchar](20),
		[pnl] [decimal](19, 6) NULL,
		[fees] [decimal](19, 6) NULL,
		[nav] [decimal](19, 6) NULL,
		[record_type] [int] NULL
		);

		create table #TempFUContribSummary(
		[date] [datetime],
		[eq_netpnl] [decimal](19, 6) NULL,
		[fi_netpnl] [decimal](19, 6) NULL,
		[cmd_netpnl] [decimal](19, 6) NULL,
		[total_netpnl] [decimal](19, 6) NULL,
		[fees] [decimal](19, 6) NULL,
		[bgn_base] [decimal](19, 6) NULL,
		[end_base] [decimal](19, 6) NULL,
		[eqrtn] [decimal](19, 10) NULL,
		[firtn] [decimal](19, 10) NULL,
		[cmdrtn] [decimal](19, 10) NULL,
		[totalrtn] [decimal](19, 10) NULL,
		[record_type] [int] NULL
		);

		create table #TempFUNotionalStaging0(
		[reval_date] [datetime],
		[pnl] [decimal](19, 6) null
		)
		;
	
		create table #TempFUNotionalStaging1(
		[reval_date] [datetime],
		[pnl] [decimal](19, 6) null,
		[nav] [decimal](19, 6) null
		)
		;

		create table #TempFUNotionalStaging2(
		[reval_date] [datetime],
		[pnl] [decimal](19, 6) null,
		[nav] [decimal](19, 6) null
		)
		;


	If  @RevalDateStart > '2016-09-30'
	BEGIN


		-- Identify futures portfolio
		if @Port = 'portcode_A'

		begin
			Set @Portfolio = @portcode_A
		end
	
		else if @Port = 'portcode_B'

		begin
			Set @Portfolio = @portcode_B
		end

		;


		 --Retieve non-UT portfolios
		WHILE DATEDIFF(DAY,@RevalDate,@RevalDateEnd)>=0
		Begin
	
		   insert into #TempFURtnsSummary 
				 (reval_date)
		   select @RevalDate 'reval_date'
	   
    
	
		-- Insert Futures PnL
		insert into #TempFURtnsSummary
			(reval_date, asset_class, pnl, fees, record_type)
		select contract_date, asset_class, sum(pnl), sum(fees), 1 'record_type'
		from (
			select	contract_date, left(security_code,1) + 'FU' 'asset_class' 
					,-isnull(total_cost_proceed_pccy, 0) 'pnl'
					,-isnull(clearing , 0)*dbo.GetCrossFxSpot(currency_1,'sgd',contract_date) 'fees'
			from	variation_margin_generation
			where	portfolio_code in (select distinct strvalue from stringtotable(',', @Portfolio))
			and		category = 'fu'
			--and		contract_date between @RevalDate and @RevalDateEnd
			and contract_date = @RevalDate
		
			union all

			select	contract_date, left(security_code,1) + 'FU' 'asset_class'
					,-isnull(total_cost_proceed_pccy, 0) 'pnl'
					,-isnull(clearing , 0)*dbo.GetCrossFxSpot(currency_1,'sgd',contract_date) 'fees'
			from	transaction_master
			where	portfolio_code in (select distinct strvalue from stringtotable(',', @Portfolio))
			--and		contract_date between @RevalDate and @RevalDateEnd
			and contract_date = @RevalDate
			and		category = 'fu'

		) combined 
		group by contract_date, asset_class
		;
	
		SET @RevalDate = DATEADD(DAY,1,@RevalDate)
		End

		insert into #TempFUContribSummary 
			(date, total_netpnl, fees, record_type)
		select		reval_date , sum(pnl), sum(fees), 1
		from		#TempFURtnsSummary
		group by	Reval_Date
		order by	Reval_Date
	

		insert into #TempFUNotionalStaging0 (reval_date, pnl)
			 Values
				('2016-09-30', 200000000)
		;

		insert into #TempFUNotionalStaging0 
			  (reval_date, pnl)
		select date, total_netpnl from #TempFUContribSummary
		order by date     
		;

		insert into #TempFUNotionalStaging0 
			  (reval_date, pnl)
		select Field_1_Value, Field_3_Value from MOS.dbo.Mapping map 
		where Field_1_Value between '2016-09-30' and @RevalDateEnd
		and Field_2_Value = 'TAA2'
		and remarks <> 'Start'
		order by Field_1_Value     
		;

		insert into #TempFUNotionalStaging1 
			  (reval_date, pnl)
		select reval_date, sum(pnl) from #TempFUNotionalStaging0 group by reval_date

		insert into #TempFUNotionalStaging2 (reval_date, pnl, nav) 		   
			 select dateadd("d", 1, reval_date) 'reval_date', pnl, 
				  nav = SUM(pnl) OVER (ORDER BY reval_date ROWS UNBOUNDED PRECEDING)
				  FROM #TempFUNotionalStaging1 
				  ORDER BY reval_date
		;
   
   
		update #TempFUContribSummary		   
			 set #TempFUContribSummary.bgn_base = two.nav
				from #TempFUContribSummary  one left outer join #TempFUNotionalStaging2 two on one.date = two.reval_date       
		;

		update		#TempFUContribSummary
		set			eq_netpnl = a1.pnl,
					eqrtn = a1.pnl/bgn_base
		from		#TempFURtnsSummary a1
		where		date = a1.reval_date
		and			a1.asset_class = 'efu'
		;


		update		#TempFUContribSummary
		set			fi_netpnl = a1.pnl,
					firtn = a1.pnl/bgn_base
		from		#TempFURtnsSummary a1
		where		date = a1.reval_date
		and			a1.asset_class = 'bfu'
		;


		update		#TempFUContribSummary
		set			cmd_netpnl = a1.pnl,
					cmdrtn = a1.pnl/bgn_base
		from		#TempFURtnsSummary a1
		where		date = a1.reval_date
		and			a1.asset_class = 'cfu'
		;


		update		#TempFUContribSummary
		set			totalrtn = total_netpnl/bgn_base
		;

		update      #TempFUContribSummary
		set         end_base = bgn_base + isnull(total_netpnl, 0)
		;


		--Insert summary row
		insert into #TempFUContribSummary 
			(date, eq_netpnl, fi_netpnl, cmd_netpnl, total_netpnl, fees, eqrtn, firtn, cmdrtn, totalrtn, record_type)
		select		@RevalDateEnd, sum(eq_netpnl) 'eq_netpnl', sum(fi_netpnl) 'fi_netpnl', sum(cmd_netpnl) 'cmd_netpnl', 
					sum(total_netpnl) 'total_netpnl', sum(fees) 'fees'
					, (power(10.00000000000,(sum(case when eqrtn <> 0 then log10(eqrtn+1) else 0 end)))-1) 'eqrtn'
					, (power(10.00000000000,(sum(case when firtn <> 0 then log10(firtn+1) else 0 end)))-1) 'firtn'
					, (power(10.00000000000,(sum(case when cmdrtn <> 0 then log10(cmdrtn+1) else 0 end)))-1) 'cmdrtn'
					, (power(10.00000000000,(sum(case when totalrtn <> 0 then log10(totalrtn+1) else 0 end)))-1) 'totalrtn'
					, 2 record_type
		from		#TempFUContribSummary
		where date between @RevalDateStart and @RevalDateEnd
		;

		update #TempFUContribSummary 
		set bgn_base = case when record_type = 1 then #TempFUContribSummary.bgn_base else a2.bgn_base end
		from (select bgn_base 'bgn_base' from #TempFUContribSummary
		where date = @RevalDateStart and record_type = 1) a2
	
		update #TempFUContribSummary 
		set end_base = case when record_type = 1 then #TempFUContribSummary.end_base else a2.end_base end
		from (select end_base 'end_base' from #TempFUContribSummary
		where date = @RevalDateEnd and record_type = 1) a2


		select	date, isnull(eq_netpnl,0) 'eq_netpnl', isnull(fi_netpnl,0) 'fi_netpnl', isnull(cmd_netpnl,0) 'cmd_netpnl'
				,isnull(total_netpnl,0) 'total_netpnl', isnull(fees,0) 'fees', isnull(bgn_base,0) 'bgn_base', isnull(end_base,0) 'end_base', isnull(eqrtn,0) 'eqrtn'
				,isnull(firtn,0) 'firtn', isnull(cmdrtn,0) 'cmdrtn', isnull(totalrtn,0) 'totalrtn', record_type
		from #TempFUContribSummary where date between @RevalDateStart and @RevalDateEnd
		order by date, record_type



	end

	end








