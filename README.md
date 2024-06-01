CREATE OR REPLACE FUNCTION pg_temp.decode_url_part(p varchar) RETURNS varchar AS $$
SELECT
	convert_from(
	CAST(E'\\x' || string_agg(
		CASE WHEN length(r.m[1]) = 1 THEN encode(convert_to(r.m[1], 'SQL_ASCII'), 'hex')
		ELSE substring(r.m[1] from 2 for 2) END, '') AS bytea)
	, 'UTF8')
FROM regexp_matches($1, '%[0-9a-f][0-9a-f]|.', 'gi') AS r(m);
$$ LANGUAGE SQL IMMUTABLE STRICT;
with all_campaigns_data as(
select 		
		ad_date,
       url_parameters,
       COALESCE(spend, 0) AS spend,
       COALESCE(impressions, 0) AS impressions,
       COALESCE(reach, 0) AS reach,
       COALESCE(clicks, 0) AS clicks,
       COALESCE(leads, 0) AS leads,
       COALESCE(value, 0) AS value,
       'facebook' as media_source
from facebook_ads_basic_daily fabd
	left join facebook_adset fa on fa.adset_id  = fabd.adset_id
	left join facebook_campaign fc on fc.campaign_id  = fabd.campaign_id
	union all
	select
		ad_date,
       url_parameters,
       COALESCE(spend, 0) AS spend,
       COALESCE(impressions, 0) AS impressions,
       COALESCE(reach, 0) AS reach,
       COALESCE(clicks, 0) AS clicks,
       COALESCE(leads, 0) AS leads,
       COALESCE(value, 0) AS value,
       'google' as media_source
   from google_ads_basic_daily gabd
)
select
	ad_date, 
	case
		when lower(substring(url_parameters, 'utm_campaign=([^\&]+)')) != 'nan'
		then pg_temp.decode_url_part(lower(substring(url_parameters, 'utm_campaign=([^\&]+)')))
	end as utm_campaign,
--	coalesce(lower(substring(url_parameters, 'utm_campaign=([^\&]+)')), '') as utm_campaign,
	sum(spend) as total_spend,
	sum(impressions) as total_impressions,
	sum(clicks) as total_clicks,
	sum(value) as total_value,
	case
   	when sum (clicks) > 0 then  round(sum (spend) / sum (clicks) :: numeric, 2)
   end as "CPC",
   case
   	when sum (impressions) > 0 then round(sum (spend) *1000 / sum (impressions) :: numeric, 2)
   end as "CPM",
   case
   	when sum (impressions) > 0 then round(sum (clicks) *100 / sum (impressions) :: numeric, 2)
   end as "CTR",
	case
   	when sum (spend) > 0 then round((sum (value) - sum (spend)) *100 / sum (spend) :: numeric, 2)
   end as "ROMI"
from all_campaigns_data
group by 1,2;



