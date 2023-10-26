/* 
   Cleaning process:
   - Drop mid_init column
   - Drop payroll number column
   - Rename 'leave_status_as_of_june_thirty' to 'leave_status'
   - Update 'work_location_borough' records
   - Update 'agency_name' records
   - The 'work_location_borough' includes locations outside of NYC 
	* Change the column name to 'work_location'
   - Remove locations outside of NYC boroughs
*/


-- Drop mid_init since middle initial will not be used
alter table city_payroll
drop column mid_init



-- This column will be unnecessary for our analyses
alter table city_payroll
drop column payroll_number


-- Simplifies column name
alter table city_payroll
rename column leave_status_as_of_june_thirty to leave_status



-- Some boroughs are labeled in lowercase format, change it to upper to keep constistency
update city_payroll 
   set work_location_borough = upper(work_location_borough)
   -- Use upper() to change strings into uppercase
	where 
	     work_location_borough ilike 'Bronx' 
	  or work_location_borough ilike 'Manhattan'
	  or work_location_borough ilike 'Queens' 
 	  or work_location_borough ilike 'Richmond'


-- Police Department agency has some records in lowercase, change it to upper to keep constistency
update city_payroll
   set agency_name = upper(agency_name)
	where
	   agency_name like 'Police Department'
			

-- Rename 'work_location_borough' to 'work_location'
alter table city_payroll
rename column work_location_borough to work_location


-- Remove locations outside of NYC boroughs
delete from city_payroll 
    where
          work_location not like 'BROOKLYN'
      and work_location not like 'BRONX'
      and work_location not like 'MANHATTAN'
      and work_location not like 'QUEENS'
      and work_location not like 'RICHMOND'



/*
    Salary Across Agencies:
    - Analyze how payroll expenses for the largest departments have changed over time since 2015.
    - Investigate the pay disparities across entry, mid, mid-senior, senior level employees
*/ 


-- Find the largest agencies by agency size in 2022
select
    c.agency_name
  , count(*) as employee_count
from
    city_payroll c
where
    c.fiscal_year = 2022 -- View values in 2022
group by
    c.agency_name
order by
    count(*) desc -- Order by largest agencies descending
limit 10 -- Only view the top 10 agencies


/* 
    Note the largest agencies:
    "DEPT OF ED PEDAGOGICAL", "DEPT OF ED PER SESSION TEACHER", "POLICE DEPARTMENT", 
    "DEPT OF ED PARA PROFESSIONALS","BOARD OF ELECTION POLL WORKERS", "DEPT OF ED HRLY SUPPORT STAFF", 
    "FIRE DEPARTMENT", "DEPARTMENT OF EDUCATION ADMIN", "DEPT OF PARKS & RECREATION", "HRA/DEPT OF SOCIAL SERVICES"
*/ 


/*
    View displays the average gross growth from 2015-2022 across 10 largest agencies. 
    We can use it to find the year-over-year difference by using 
    a simple calculation involving the lag function. However, the agencies
    'DEPT OF ED PER SESSION TEACHER', 'BOARD OF ELECTION POLL WORKERS', and
    'DEPT OF ED HRLY SUPPORT STAFF do not include values for 'base_salary' column, instead
     we will use 'regular_gross_paid' column to determine the yearly income for those agencies.
*/ 
with largest_agencies as (
	select 
	    c.agency_name
	  , c.fiscal_year
	  , avg(c.base_salary) as avg_yearly_income
	from
	    city_payroll c
	where
	    c.agency_name in (
		'DEPT OF ED PEDAGOGICAL', 
		'POLICE DEPARTMENT', 'DEPT OF ED PARA PROFESSIONALS',
		'FIRE DEPARTMENT', 'DEPARTMENT OF EDUCATION ADMIN', 
		'DEPT OF PARKS & RECREATION', 'HRA/DEPT OF SOCIAL SERVICES')
	group by
	    c.agency_name
	  , c.fiscal_year

	union  -- Combine results with the following query

	select 
	    c.agency_name
	  , c.fiscal_year
	  , avg(c.regular_gross_paid) as avg_yearly_income
	from
	    city_payroll c
	where
	    c.agency_name in (
		'DEPT OF ED PER SESSION TEACHER', 
		'BOARD OF ELECTION POLL WORKERS', 
		'DEPT OF ED HRLY SUPPORT STAFF')
	group by
	    c.agency_name,
	    c.fiscal_year
	order by
	    agency_name
	  , fiscal_year
)


-- Use lag function to find year over year difference of pay roll expenses
select
    a.agency_name
  , a.fiscal_year
  , a.avg_yearly_income
  , a.avg_yearly_income - lag(a.avg_yearly_income) over(partition by a.agency_name) AS yoy_difference
    /* 	The lag function will obtain the value of the column
	avg_gross for the previous record. Then, the calculation
	finds the difference between the avg gross of the year in the record
	and the avg gross of the previous record to find the 
	year-over-year difference. */
from
    largest_agencies a
order by
    a.agency_name asc -- Order by agency_name in alphabetical order
  , a.fiscal_year asc -- Order by oldest year to recent year

/*
    The agencies that have experienced average gross growth
    and the ones that have remained stable or declined are in
    the documentation.
 */
 

-- Compare the compensation of entry, mid, mid-senior, and senior level employees in the largest agencies.


/*
     CTE displaying the employee name, when they started working,
     their tenure, and their avg yearly income. Join this CTE to the 
     largest_agencies view created earlier to accurately display
     yearly_income. Includes a case statement that takes the average 
     of'regular_gross_paid' column for 'DEPT OF ED PER SESSION TEACHER',
     'BOARD OF ELECTION POLL WORKERS', and 'DEPT OF ED HRLY SUPPORT STAFF' 
     since these agencies do not include records from the 'base_salary' column, 
*/
with employee_tenure_and_income as  (
	select
	    concat(c.first_name, ' ', c.last_name) as employee_name -- Combine first and last name
	  , c.agency_start_date
	  , ('2023-03-15' - c.agency_start_date) as days_employed -- Data was last updated on '2023-03-15' so it will be used a 'current_date'
	  , case
		when c.agency_name in (
			'DEPT OF ED PER SESSION TEACHER', 
			'BOARD OF ELECTION POLL WORKERS', 
			'DEPT OF ED HRLY SUPPORT STAFF') then avg(c.regular_gross_paid)  -- Find avg yearly income of the employee
		else avg(c.base_salary)
		end as yearly_income
	from
	    city_payroll c
	where
	    c.leave_status like 'ACTIVE' -- Only look at active employees
	group by
	    employee_name
	  , c.agency_start_date
	  , c.agency_name
)


/*
    CTE that gives each employee a 'level' based on conditions given in 
    case statement that determines if they are entry, mid, mid-senior, or senior level 
    employees
*/
, employee_income_and_level as (
	select
	    e.employee_name
	  , e.days_employed
	  , e.yearly_income
	  , case
		when e.days_employed <= 1095 then 'Entry level' -- 1-3 years for entry level
		when e.days_employed > 1095 and e.days_employed <= 2190 then 'Mid level' -- 3-6 years for mid level
		when e.days_employed > 2190 and e.days_employed <= 3285 then 'Mid-senior' -- 6-9 years for mid-senior levle
		when e.days_employed >= 3285 then 'Senior level' -- 9+ years for senior level
	    else 'other' 
	    end as employee_level
	from
	    employee_tenure_and_income e
	where
	    yearly_income > 0 -- Do not include negative incomes or incomes equal to 0

)


/*
     Display a statistical summary of each employee level
     that includes the avg salary, the min, and the max salary
*/
select
    e.employee_level
  , round(avg(e.yearly_income),2) as avg_yearly_income
  , round(min(e.yearly_income),2) as min_yearly_income
  , round(max(e.yearly_income),2) as max_yearly_income
from
    employee_income_and_level e
where
    e.employee_level <> 'other'
group by
    e.employee_level
order by
    avg(e.yearly_income) asc -- Order by level with least avg yearly income
	
/*
    We can use the results of our output to discover disparities of pay among the levels
    within our documentation
*/

/*
    Budget Analysis:
    - Analyze historic data of budgeting across the agencies with the most employees
    - Investigate which departments on average require the most budgeting, based on the 
      data. I hypothesize that agencies with higher employee counts require more budgeting
    - In 2022, investigate which of the departments were over budget or under budget
      to project budgeting needs in 2023
    - See which departments rely heavily on overtime payments. 
*/


-- Analyze historic data of budgeting across the agencies with the most employees


-- Find the largest agencies by employee count
select
    c.agency_name
  , count(*) as employee_count
from
    city_payroll c
group by
    c.agency_name
order by
    count(*) desc -- Order by the agencies with largest employee count
limit 10 -- Only view the top 10 agencies


/*
	Display a list of the most budgeted agencies, and their base budget 
	for the years 2015 - 2022
*/
with largest_agencies as (
	select 
	    c.agency_name
	  , c.fiscal_year
	  , count(*) as employee_count -- View employee count of agency
	  , sum(c.base_salary) as base_budget -- Total up the base budget of that year
	from
	    city_payroll c
	where
	    c.agency_name in (
		'DEPT OF ED PEDAGOGICAL', 
		'POLICE DEPARTMENT', 'DEPT OF ED PARA PROFESSIONALS',
		'FIRE DEPARTMENT', 'DEPARTMENT OF EDUCATION ADMIN', 
		'DEPT OF PARKS & RECREATION', 'HRA/DEPT OF SOCIAL SERVICES'
		'DEPT OF ED PER SESSION TEACHER', 'BOARD OF ELECTION POLL WORKERS', 
		'DEPT OF ED HRLY SUPPORT STAFF')
	
	    and c.leave_status like 'ACTIVE'
	group by
	    c.agency_name
	  , c.fiscal_year
	order by
	    c.agency_name asc
      	  , c.fiscal_year

)


-- Determine which agencies need the most budgeting each year on average
, avg_budget_allocations as (
	select
	    a.agency_name 
	  , avg(a.employee_count) as avg_employee_count -- Find the average employee count 
	  , avg(a.base_budget) as avg_base_budget -- Find average budget
	from
	    largest_agencies a
	group by
	    a.agency_name
	order by
	    avg(a.base_budget) desc -- Order by the agencies with the most budget historically

)


-- See if higher employee counts correlate with larger budgets for salaries
select
    corr(a.avg_employee_count, a.avg_base_budget)
from
    avg_budget_allocations a


-- 0.91: high correlation between employee count and budget
	

-- In 2022, investigate which of the departments were over budget or under budget


/*
    CTE displaying the agency, their employee count,
    their estimated budget of 2022, and the actual spent
    in 2022. Compare the budget vs actual to see if the 
    agency is over or under budget.
*/
with budget_vs_actual as (	
	select
	    c.agency_name
	  , count(*) as employee_count -- Count employees in agencies
	  , sum(c.base_salary) as salary_budget -- Total up estimated salary budget
	  , sum(c.regular_gross_paid) as actual_spent -- Total up actual salary budget spent
	from
	    city_payroll c
	where
	        c.fiscal_year = 2022 -- Only view values in 2022
	    and c.leave_status like 'ACTIVE' -- Only view active employees
	group by	
	    c.agency_name
	order by
	    count(*) desc -- Order by most employee count
	limit 10 -- View top 10 agencies by the most employees
	
	
)


-- Investigate which agencies are under or over budget
select
    b.agency_name
  , b.salary_budget
  , b.actual_spent
  , (b.salary_budget - b.actual_spent) as remaining_budget
   -- Subtract actual_spent by salary_budget to see remaining budget
from
    budget_vs_actual b

-- Findings are documented in documentation and dashboard
	

-- See which departments rely heavily on overtime payments. 


/*
    CTE displaying every agency in 2022 that includes their employee count,
    avg ot hours per employee total employee gross, total overtime gross, 
    and a calculation that finds out the percentage in which overtime 
    payments take up total yearly gross.
*/
with ot_agencies as (
	select
	    c.agency_name
	  , count(*) as employee_count -- Count employees are employed
	  , avg(c.ot_hours) as avg_ot_hours -- Avg ot hours per employee in agency 
	  , (sum(distinct(c.regular_gross_paid)) + sum(distinct(c.total_other_pay))) as total_gross
		-- Add total_other_pay to regular_gross_paid to find accurate funding to workers 
	  , sum(distinct(c.total_ot_paid)) as total_ot_gross -- Calculate total ot gross 
	from
	    city_payroll c
	where
	        c.fiscal_year = 2022 -- Only view values in 2022
	    and c.leave_status like 'ACTIVE' -- Only view active employees
	group by
	    c.agency_name
	order by
	    avg(c.ot_hours) desc -- Order by the agencies with most avg ot hours per agency
)


-- Output the results to find which agencies rely on overtime payments
select
    t.agency_name
  , t.employee_count
  , t.avg_ot_hours
  , t.total_gross
  , t.total_ot_gross
  , ((t.total_ot_gross / t.total_gross) * 100) as ot_percentage_of_income
     /* This calculation divides total_ot_gross by total_gross multipled by 100
	to get the percentage overtime pay takes up of total gross of 2022*/
from
    ot_agencies t
order by 
    ((t.total_ot_gross / t.total_gross) * 100) desc


/*
    We can analyze these overtime results further in the documentation
    and determine how to contribute to better resource management, 
    employee well-being, and overall efficiency in city operations for 2023.
*/



/*	
    Hourly Wage Analysis: 
    - Calculate the percentiles of hourly wage of the largest agencies and see if
      lower percentiles typically have more overtime hours than top earners on 
      average.
    - Identify factors that influence hourly wages. You can consider 
      factors like agency size, location, or job role as independent variables.
    - Calculate the avg hours and ot hours worked by employees agency over time. 
      This can help you understand which areas require the most labor.
*/


/*
    Calculate the percentiles of hourly wage of the largest agencies and see if
    lower percentiles typically have more overtime hours than top earners on 
    average.
*/


/*
    CTE displaying each employee who is active in the largest agencies,
    their name, the agency they work for, their hourly wage, and overtime hours in 2022.
    I hypothesize that workers in the 25th percentile of hourly wages will typically have
    much more overtime hours on average than those in the 75th, and 95th percentiles.
*/
with employee_hrly_wage as (
	select
	    c.agency_name
	  , concat(c.first_name, ' ', c.last_name) as employee_name -- Combine employee name
	  , case 
		when c.agency_name ilike 'DEPT OF ED PARA PROFESSIONALS' 
		and c.agency_name ilike 'DEPT OF ED PEDAGOGICAL'
		then (c.regular_gross_paid / 2200) -- Divide regular gross paid by estimated hours worked by teachers
		else (c.regular_gross_paid / c.regular_hours) -- Divide regular_gross_paid by regular_hours to get hourly wage
	    end as hrly_wage 
	 , c.ot_hours as ot_hours -- Find ot hours of each worker in 2022
	from
	    city_payroll c
	where
	    c.agency_name in (
		'DEPT OF ED PEDAGOGICAL', 'DEPT OF ED PER SESSION TEACHER', 
		'POLICE DEPARTMENT', 'DEPT OF ED PARA PROFESSIONALS',
		'BOARD OF ELECTION POLL WORKERS', 'DEPT OF ED HRLY SUPPORT STAFF', 
		'FIRE DEPARTMENT', 'DEPARTMENT OF EDUCATION ADMIN', 
		'DEPT OF PARKS & RECREATION', 'HRA/DEPT OF SOCIAL SERVICES')
		
	    and c.fiscal_year = 2022 -- Only view values in 2022
	    and c.regular_hours > 0 -- Only filter employees with more than 0 zeros
	    and c.leave_status like 'ACTIVE' -- Active employees only
	group by
	    c.agency_name
	  , c.first_name
	  , c.last_name
	  , c.regular_gross_paid
	  , c.regular_hours
	  , c.ot_hours
)


-- Calculate the percentiles of hourly wage in the agencies
, hrly_wage_percentile as ( 
	select
	    e.agency_name
	  , e.hrly_wage
	  , e.ot_hours
	  , ntile(100) over(partition by e.agency_name order by e.hrly_wage) as percentile
	    -- Use 'ntile' function to find the percentiles of each employee's hourly wage
	from
	    employee_hrly_wage e

)


-- Output results of hourly wage and ot hour percentiles of each agency
, hrly_wage_percentiles as (	
	select
	    h.agency_name
	  , round(avg(h.hrly_wage),2) as hrly_wage
	    -- Avg the hourly wage of each employee that is in the 25th, 50th, 75th, and 100th percentile
	  , round(avg(h.ot_hours),2) as ot_hours -- View avg ot hours of each employee that is in those percentiles
	  , h.percentile
	from
	    hrly_wage_percentile h
	where
	      h.percentile = 25 -- 25th percentile
	   or h.percentile = 50 -- Median
	   or h.percentile = 75 -- 75th percentile
	   or h.percentile = 95 -- Top earners
	group by
	    h.agency_name
	  , h.percentile
	order by
	     h.agency_name asc -- Order by agency name alphabetically
	   , h.percentile asc -- Order by percentile from 25th onwards

)


-- Output the results and see if the lowest earners of hourly wages have more OT hours worked on average
select
    h.percentile as hrly_wage_percentile
  , avg(h.ot_hours) as avg_ot_hours -- Avg OT hours of each percentile
from
    hrly_wage_percentiles h 
group by
    hrly_wage_percentile

		 
-- It appears that the hypothisis was correct and the 25th percentile of hourly wages typically have the most OT hours worked

	
/*
    Identify factors that influence hourly wages. You can consider 
    factors like agency size, location, cost of living as variables
*/


-- Output every agency, their active employees, and the average hourly wage per employee
with agency_size_and_wage as (
	select
	    c.agency_name
	  , count(distinct(c.first_name)) as employee_count
	  , round(avg(c.regular_gross_paid / c.regular_hours),2) as hrly_wage 
	from
	    city_payroll c
	where
	      c.leave_status like 'ACTIVE'
	  and c.regular_hours > 0
	group by
	    c.agency_name
	order by
	    count(distinct(c.first_name)) desc

)

select
    corr(a.employee_count, a.hrly_wage) as correlation -- Determine correlation between employee count and hourly wage
from
    agency_size_and_wage a
	
-- No correlation between agency size and hourly wage
	

-- See correlation between hourly wage and location

/*
    CTE displays the borough, its population, the avg cost of a home,
    and the average hourly wage per borough
*/
with borough_and_hrly_wage as (
	select
	    n.work_location
	  , n.pop_size
	  , n.avg_cost_of_home
	  , round(avg(c.regular_gross_paid / c.regular_hours),2) as hrly_wage
	from
	    city_payroll c
		join nyc_boroughs n
		   on c.work_location = n.work_location
	where
	       c.leave_status like 'ACTIVE'
	   and c.regular_hours > 0
	group by
	    n.work_location
	  , n.pop_size
	  , n.avg_cost_of_home
	order by
	    round(avg(c.regular_gross_paid / c.regular_hours),2) desc -- Order by most hourly wage

)

-- Find correlation between avg cost of home within the borough, and pop size of the borough with hourly wage
select
    corr(b.avg_cost_of_home, b.hrly_wage) -- Change variable to pop size to see other results
from
    borough_and_hrly_wage b
	
-- Positive correlation between avg cost of home and hourly wage
-- Mid correlation between pop size and hourly wage

	
/*
    Calculate the avg hours and ot hours worked by agency over time. 
    This can help us understand which areas require the most labor.
*/


-- Find avg hours (including ot hours) worked by employees in each agency over 2015-2022
with avg_total_hrs_worked_by_agency as (
	select
	    c.fiscal_year
	  , c.agency_name
	  , avg(c.regular_hours + c.ot_hours) as avg_hrs_worked
	    -- Add regular hours and ot hours worked, then find the average of that
	from
	    city_payroll c
	where
	    c.agency_name in (
			'DEPT OF ED PEDAGOGICAL', 'DEPT OF ED PER SESSION TEACHER', 
			'POLICE DEPARTMENT', 'DEPT OF ED PARA PROFESSIONALS',
			'BOARD OF ELECTION POLL WORKERS', 'DEPT OF ED HRLY SUPPORT STAFF', 
			'FIRE DEPARTMENT', 'DEPARTMENT OF EDUCATION ADMIN', 
			'DEPT OF PARKS & RECREATION', 'HRA/DEPT OF SOCIAL SERVICES')

	    and c.fiscal_year between '2015' and '2022' -- No values passed 2022
	group by
	    c.agency_name 
	  , c.fiscal_year 
	having
	    avg(c.regular_hours + c.ot_hours) > 0
	order by
	    c.agency_name asc -- Order by agency name alphabetically
	  , c.fiscal_year asc -- Order by fiscal year from 2015 onwards

)


/*
    Output the agencies with the most avg hours worked every year from 2015-2022
    and using 'dense_rank', rank the agencies by the most hours worked to the least
    per year.
*/
, most_hrs_worked_by_year as (
	select
	    a.fiscal_year
	  , a.agency_name
	  , a.avg_hrs_worked
	  , dense_rank() over(partition by a.fiscal_year order by a.avg_hrs_worked desc) as rank_val
	    -- Rank the agencies by the most avg hours worked to the least, partition them into each year
	from
	    avg_total_hrs_worked_by_agency a
	group by
	    a.fiscal_year
	  , a.agency_name
	  , a.avg_hrs_worked
	order by
	    a.fiscal_year asc -- Order values by ascending years (2015 - 2022)

)


-- Output how many times an agency was ranked 1st for the most hours worked from 2015 - 2022
select
    m.agency_name
  , count(*) as agency_occurences 
    -- Count how many times an agency had the most avg hours worked from 2015-2022
from
    most_hrs_worked_by_year m
where
    rank_val = 1 -- View agencies that were only ranked first
group by	
    m.agency_name
	
		 
-- The Fire Department was ranked 1st for every year in the msot hours worked 

	
/*
    NYC Borough Analysis:
    - Which boroughs make more money on average than others?
    - Correlations between payroll expenses and other factors, 
      such as pop. size, crime rates, cost of living within each borough. 
*/


-- Find which boroughs have the most avg yearly income 
select
  , c.work_location
  , avg(c.base_salary) as avg_yearly_income -- Find average base salary of every employee in the boroughs
from
    city_payroll c
group by
    c.work_location
order by
    avg(c.base_salary) desc -- Order by borough that has the highest average yearly income


-- Find the running 4 year average yearly gross in the boroughs
with avg_gross_per_borough as (
	select
	    c.fiscal_year
	  , c.work_location
	  , avg(c.regular_gross_paid) as avg_yearly_gross -- Avg yearly gross of the borough
	from
	    city_payroll c
	group by
	    c.fiscal_year
	  , c.work_location
	order by 
	    c.work_location asc
	  , c.fiscal_year asc
	  
)


select
    a.fiscal_year
  , a.work_location
  , a.avg_yearly_gross
  , avg(a.avg_yearly_gross) over(partition by work_location order by a.fiscal_year rows between 3 preceding and current row)
    as avg_gross_trailing_4y -- Use window function to find rolling average of 4 years and partitioned by the work location
from
    avg_gross_per_borough a 
order by    
   a.work_location asc
   
   
/*
    Using the rolling averages, we can find long-term trends and
    determine the factors that cause these trends in the
    flunctuation of avg gross between boroughs.
*/


/*
    Employment and Tenure Analysis:
    - Determine what months in the data typically have the most hires on average and why.
    - Determine which of the largest agencies have the most active, ceased employees
    - Analyze how employee tenure influences leave usage. Do employees with longer tenures 
      take more or less leave compared to newer employees?
*/


-- Determine what months in the data typically have the most hires on average and why.


/*
    Add the first day of the month column of the date they hired on,
    that way we can summarize how many employees were hired in every month
    from every year in the data.
*/
with first_day_of_month as (
	select
	    c.*
	  , cast(date_trunc('month', c.agency_start_date) as date) as first_day
	    -- Use date_trunc to the get the first day of the month, cast it as a date
	from
	   city_payroll c

)


-- Create historical summary of how many hires took place in each month of every year 
, monthly_summary_of_hires as (
	select
	    f.first_day
	  , count(*) as total_hires_per_month -- Count how many hires there were in each month
	from 
	    first_day_of_month f
	group by	
	    f.first_day -- Group the values by the corresponding month in the year

)


-- Output results for the months with most hires per month
select
    date_part('month', m.first_day) as months -- Part out month value from first_day column
  , round(avg(m.total_hires_per_month),0) as avg_hires_per_month -- Find avg hires per month historically
from
    monthly_summary_of_hires m
group by
    months
order by
    round(avg(m.total_hires_per_month),0) desc -- Order output by most avg hires per month
	
	
-- January and September has the most hires on average out of every month


-- Determine which of the largest agencies have the most active, and ceased employees

		 
-- Output the agencies with the most active employees and their avg hours worked
select 
    c.agency_name
  , avg(c.regular_hours) as avg_hrs_worked
  , count(*) as active_employees
from	
    city_payroll c
where
    c.agency_name in (
		'DEPT OF ED PEDAGOGICAL', 'DEPT OF ED PER SESSION TEACHER', 
		'POLICE DEPARTMENT', 'DEPT OF ED PARA PROFESSIONALS',
		'BOARD OF ELECTION POLL WORKERS', 'DEPT OF ED HRLY SUPPORT STAFF', 
		'FIRE DEPARTMENT', 'DEPARTMENT OF EDUCATION ADMIN', 
		'DEPT OF PARKS & RECREATION', 'HRA/DEPT OF SOCIAL SERVICES')		
    and c.fiscal_year = 2022
    and c.leave_status ilike 'ACTIVE'
group by
    c.agency_name
  , c.leave_status
order by
    count(*) desc

		 
-- Output the agencies with the most ceased employees in 2022
select 
    c.agency_name
  , avg(c.regular_hours) as avg_hrs_worked
  , count(*) as ceased_employees
from	
    city_payroll c
where
    c.agency_name in (
		'DEPT OF ED PEDAGOGICAL', 'DEPT OF ED PER SESSION TEACHER', 
		'POLICE DEPARTMENT', 'DEPT OF ED PARA PROFESSIONALS',
		'BOARD OF ELECTION POLL WORKERS', 'DEPT OF ED HRLY SUPPORT STAFF', 
		'FIRE DEPARTMENT', 'DEPARTMENT OF EDUCATION ADMIN', 
		'DEPT OF PARKS & RECREATION', 'HRA/DEPT OF SOCIAL SERVICES')		
    and c.fiscal_year = 2022
    and c.leave_status ilike 'CEASED'
group by
    c.agency_name
  , c.leave_status
order by
    count(*) desc


/*
    Analyze how employee tenure influences leave usage. Do employees with longer 
    tenures take more or less leave compared to newer employees?
*/
		 

/*
    Re-use previous salary CTEs where we placed employees tenure length in certain time stamps
    in order to label them into entry, mid, mid-senior, and senior columns. Here, we will use
    the CTE to see out of all of the employee levels, which level takes more leave
    in the year of 2022.
*/
with employee_tenure as  (
	select
	    concat(c.first_name, ' ', c.last_name) as employee_name -- Combine first and last name
	  , c.agency_start_date
	  , ('2022-12-31' - c.agency_start_date) as days_employed -- Only look at values up until the end of 2022 
	  , c.leave_status
	from
	    city_payroll c
	where
	    c.agency_name in (
		'DEPT OF ED PEDAGOGICAL', 'DEPT OF ED PER SESSION TEACHER', 
		'POLICE DEPARTMENT', 'DEPT OF ED PARA PROFESSIONALS',
		'BOARD OF ELECTION POLL WORKERS', 'DEPT OF ED HRLY SUPPORT STAFF', 
		'FIRE DEPARTMENT', 'DEPARTMENT OF EDUCATION ADMIN', 
		'DEPT OF PARKS & RECREATION', 'HRA/DEPT OF SOCIAL SERVICES') -- Look at the largest agencies
	
	    and c.leave_status ilike 'ON LEAVE' -- Only look at active employees
	    and c.fiscal_year = 2022 -- Values in 2022
	group by
	    employee_name
	  , c.agency_start_date
	  , c.agency_name
	  , c.leave_status
)


/*
    CTE that gives each employee a 'level' based on conditions given in 
    case statement that determines if they are entry, mid, mid-senior, or senior level 
    employees
*/
, employee_tenure_and_level as (
	select
	    e.employee_name
	  , e.days_employed
	  , case
		when e.days_employed <= 1095 then 'Entry level' -- 1-3 years for entry level
		when e.days_employed > 1095 and e.days_employed <= 2190 then 'Mid level' -- 3-6 years for mid level
		when e.days_employed > 2190 and e.days_employed <= 3285 then 'Mid-senior' -- 6-9 years for mid-senior levle
		when e.days_employed >= 3285 then 'Senior level' -- 9+ years for senior level
	    else 'other' 
	    end as employee_level
	from
	    employee_tenure e

)


select 
    e.employee_level -- Group the output by entry, mid, mid-senior, and senior levels
  , count(*) as leave_rate -- Count how many employees are on leave from each level in the year 2022
from
    employee_tenure_and_level e
group by
    e.employee_level

-- As expected, senior level workers took the most leave in 2022
	
