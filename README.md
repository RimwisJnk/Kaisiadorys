----------------------------------/> 2015  VMS+BIBA III K. IVYKDYMAS EUR</--------------------------------------------------
select
---b.budget_version_id,
 b.budget_name Versija,
 b.segment1 Istaiga,
 b.description Istaigos_pav,
 b.segment3 fin_saltinis,
 b.segment4 programa,
 LPAD(b.segment4, 2) prg,
 b.segment5 funk_klas,
 LPAD(b.segment5, 2) funk_k,
 b.segment6 ekon_klas,
 b.segment7 samata,
 b.segment8 AV,
 to_number(b.met_suma_b) Skirta_biudzeto,
 to_number(p.suma_p) Pateikta_paraisku,
 to_number(g.suma_G) gauti_asign,
 to_number(k.suma_k) Kasines_islaidos
--g.suma_G - k.suma_k likutis_sask,
--p.suma_p - g.suma_G negauta_asign,
--b.met_suma_b - p.suma_p asign_atemus_par

       from

-- BIUDÞETO SUMOS ---- biudzetas traukiamas is VMAS DK knygos
(select
       bv.latest_opened_year,
       ver.budget_version_id,
       ver.budget_name,
       kfv.segment1,
       valtl.description,
       kfv.segment3,
       kfv.segment4,
       kfv.segment5,

       (case
          when kfv.segment6 in ('2.2.1.1.1.3','2.2.1.1.1.13', '2.2.1.1.1.20', '2.2.1.1.1.20.1.', '2.2.1.1.1.20.2.', '2.2.1.1.1.20.3.', '2.2.1.1.1.20.4.')  then '2.2.1.1.1.20'
          when kfv.segment6 like '3%I.'  then replace(kfv.segment6, 'I.','')
          else kfv.segment6
       end)segment6,
       kfv.segment7,
       kfv.segment8,
       sum(nvl(bal.period_net_dr, 0) - nvl(bal.period_net_cr, 0)) met_suma_b,
       sum(case when per.quarter_num = 1 then nvl(bal.period_net_dr, 0) - nvl(bal.period_net_cr, 0) else 0 end) I_Ketv_b,
       sum(case when per.quarter_num = 2 then nvl(bal.period_net_dr, 0) - nvl(bal.period_net_cr, 0) else 0 end) II_Ketv_b,
       sum(case when per.quarter_num = 3 then nvl(bal.period_net_dr, 0) - nvl(bal.period_net_cr, 0) else 0 end) III_Ketv_b,
       sum(case when per.quarter_num = 4 then nvl(bal.period_net_dr, 0) - nvl(bal.period_net_cr, 0) else 0 end) IV_Ketv_b

  from gl_balances bal
  join gl_code_combinations_kfv kfv on kfv.code_combination_id = bal.code_combination_id
  join fnd_flex_values val on val.flex_value = kfv.segment1  and val.flex_value_set_id = 1009712
  join fnd_flex_values_tl valtl on valtl.flex_value_id = val.flex_value_id and valtl.language = 'LT'
  join gl_budget_versions ver on ver.budget_version_id = bal.budget_version_id
  join gl_budgets bv on bv.budget_name = ver.budget_name
  join gl_periods per on per.period_name = bal.period_name and per.period_set_name = 'VMS pagrindinis'

  where bal.set_of_books_id = 5013-- =2015 VMS DK 5013 kaupimo
  and bal.actual_flag = 'B'
  and kfv.summary_flag <> 'Y'
  and nvl(bal.period_net_dr, 0) - nvl(bal.period_net_cr, 0) <> 0
  and kfv.segment6 not like '0%'
  and kfv.segment6 not like '1%'
  and kfv.segment6 not like '%P%'
  and kfv.segment6 not like '4%'
/*  and bal.budget_version_id = 1441
  and kfv.segment1 = '8821029'
  and kfv.segment3 = '01'
  and kfv.segment4 = '09030101'
  and kfv.segment5 = '08.02.01.08.'
  and kfv.segment6 = '2.2.1.1.1.16.'
  and kfv.segment7 = '4000460'
  and kfv.segment8 = '1030000'*/

  group by
       bv.latest_opened_year,
       ver.budget_version_id,
       ver.budget_name,
       kfv.segment1,
       valtl.description,
       kfv.segment3,
       kfv.segment4,
       segment5,
       segment6,
       kfv.segment7,
       kfv.segment8
  having
       sum(case when per.quarter_num = 1 then nvl(bal.period_net_dr, 0) - nvl(bal.period_net_cr, 0) else 0 end) <> 0 or
       sum(case when per.quarter_num = 2 then nvl(bal.period_net_dr, 0) - nvl(bal.period_net_cr, 0) else 0 end) <> 0 or
       sum(case when per.quarter_num = 3 then nvl(bal.period_net_dr, 0) - nvl(bal.period_net_cr, 0) else 0 end) <> 0 or
       sum(case when per.quarter_num = 4 then nvl(bal.period_net_dr, 0) - nvl(bal.period_net_cr, 0) else 0 end) <> 0 ) b,

-- PARAIÐKØ SUMOS  --- traukiamas is DK knygos
(select
       to_char(par.ivedimo_data, 'YYYY') period_year,
       eil.ist_kodas segment1,
       par.fin_saltinio_kodas segment3,
       eil.programos_kodas segment4,
       --eil.fun_klasif_kodas segment5,
       eil.eko_klasif_kodas segment6,
       eil.samatos_kodas segment7,
       sum(eil.itraukiama_suma) suma_p

  from xx_fin_ms010_paraiskos_all par
  join xx_fin_ms010_prs_eilutes_all eil on eil.paraiskos_id = par.paraiskos_id

 where par.busenos_kodas not in ('X', 'N')
   and par.tipo_kodas not in ('1', '2', '10', '11')
   and eil.paskirst = 'N'
   and par.org_id = 1363    ---org ID 1363  Administracija EUR   -- 1350 Biud?etini¸ ?staig¸ buhalterin? apskaita EUR
   and nvl(par.isor_institucijos_id, 0) <> '101639'

group by
       to_char(par.ivedimo_data, 'YYYY'),
       eil.ist_kodas,
       par.fin_saltinio_kodas,
       eil.programos_kodas,
       --eil.fun_klasif_kodas,
       eil.eko_klasif_kodas,
       eil.samatos_kodas
having sum(eil.itraukiama_suma) <> 0) p,

-- KASINËS IÐLAIDOS --- traukiamas is VMAS ir BIBA DK pp
 (select
       y.period_year,
       y.segment1,
       y.segment3,
       y.segment4,
       y.segment6,
       y.segment7,
       sum(y.suma_k) suma_k,
       sum(y.I_Ketv_K) I_Ketv_K,
       sum(y.II_Ketv_K) II_Ketv_K,
       sum(y.III_Ketv_K) III_Ketv_K,
       sum(y.IV_Ketv_K) IV_Ketv_K

       from

(select
       k.period_year,
       k.segment1,
       k.segment3,
       k.segment4,
       k.segment6,
       k.segment7,
       sum(k.suma) suma_k,
       sum(case when k.quarter_num = 1 then nvl(k.suma, 0) else 0 end) I_Ketv_K,
       sum(case when k.quarter_num = 2 then nvl(k.suma, 0) else 0 end) II_Ketv_K,
       sum(case when k.quarter_num = 3 then nvl(k.suma, 0) else 0 end) III_Ketv_K,
       sum(case when k.quarter_num = 4 then nvl(k.suma, 0) else 0 end) IV_Ketv_K

  from xx_disc_kasines_islaidos_e1 k
  where k.set_of_books_id IN ('5014', '5001') ----VMSA  ir BIBA DK pp
/*   and k.period_year = 2011
   and k.segment7 like '4000460'
   and k.segment1 = '188710061'
   and k.segment6 = '2.8.1.1.1.2.'*/
group by
       k.period_year,
       k.segment1,
       k.segment3,
       k.segment4,
       k.segment6,
       k.segment7
having sum(k.suma) <> 0

UNION ALL

select p.period_year,
       p.par_istaigos_kodas segment1,
       p.segment3,
       p.segment4,
       p.segment6,
       p.segment7,
       sum(p.suma) suma_k,
       sum(case when p.quarter_num = 1 then nvl(p.suma, 0) else 0 end) I_Ketv_K,
       sum(case when p.quarter_num = 2 then nvl(p.suma, 0) else 0 end) II_Ketv_K,
       sum(case when p.quarter_num = 3 then nvl(p.suma, 0) else 0 end) III_Ketv_K,
       sum(case when p.quarter_num = 4 then nvl(p.suma, 0) else 0 end) IV_Ketv_K

  from xx_disc_kasines_islaidos_e2 p
  where p.set_of_books_id IN ('5014', '5001') ----VMSA  ir BIBA
/*   and p.period_year = 2011
   and p.segment7 like '4000460'
   and p.segment1 = '188710061'
   and p.segment6 = '2.8.1.1.1.2.'
*/
group by
       p.period_year,
       p.par_istaigos_kodas,
       p.segment3,
       p.segment4,
       p.segment6,
       p.segment7
having sum(p.suma) <> 0

UNION ALL

--mokëjimai pagal paraiðkas nepavaldþioms ástaigoms
select
       z.period_year,
       nvl(z.vendor_code, 0) as segment1,
       z.segment3,
       z.segment4,
       z.segment6,
       z.segment7,
       sum(z.suma) suma_k,
       sum(case when z.quarter_num = 1 then nvl(z.suma, 0) else 0 end) I_Ketv_K,
       sum(case when z.quarter_num = 2 then nvl(z.suma, 0) else 0 end) II_Ketv_K,
       sum(case when z.quarter_num = 3 then nvl(z.suma, 0) else 0 end) III_Ketv_K,
       sum(case when z.quarter_num = 4 then nvl(z.suma, 0) else 0 end) IV_Ketv_K

  from xx_disc_kasines_islaidos_e3 z
  where z.set_of_books_id IN ('5014', '5001') ----VMSA  ir BIBA
/*    and z.vendor_code <> '188710061'
    and z.period_year = 2011
    and z.segment7 like '4000460'
    and z.segment1 = '188710061'
    and z.segment6 = '2.8.1.1.1.2.'*/


group by
       z.period_year,
       z.vendor_code,
       z.segment3,
       z.segment4,
       z.segment6,
       z.segment7
having sum(z.suma) <> 0) y


group by
       y.period_year,
       y.segment1,
       y.segment3,
       y.segment4,
       y.segment6,
       y.segment7) k,

-- GAUTAS FINANSAVIMAS
(select g.period_year,
       g.par_istaigos_kodas,
       g.segment3,
       g.segment4,
       g.segment6,
       g.segment7,
       sum(g.suma) suma_G,
       sum(case when g.quarter_num = 1 then nvl(g.suma, 0) else 0 end) I_Ketv_G,
       sum(case when g.quarter_num = 2 then nvl(g.suma, 0) else 0 end) II_Ketv_G,
       sum(case when g.quarter_num = 3 then nvl(g.suma, 0) else 0 end) III_Ketv_G,
       sum(case when g.quarter_num = 4 then nvl(g.suma, 0) else 0 end) IV_Ketv_G

  from xx_disc_gautas_finansavimas_e4 g
  where g.set_of_books_id IN ('5014', '5001') ----VMSA  ir BIBA
/* where g.period_year = 2010
   and g.segment1 = '188710061'*/


group by
       g.period_year,
       g.par_istaigos_kodas,
       g.segment3,
       g.segment4,
       g.segment6,
       g.segment7
having sum(g.suma) <> 0) g


  where   b.segment1 = k.segment1(+)
  and     b.segment3 = k.segment3(+)
  and     b.segment4 = k.segment4(+)
  and     b.segment6 = k.segment6(+)
  and     b.segment7 = k.segment7(+)
  and     b.latest_opened_year = k.period_year(+)
  and     b.segment1 = g.par_istaigos_kodas(+)
  and     b.segment3 = g.segment3(+)
  and     b.segment4 = g.segment4(+)
  and     b.segment6 = g.segment6(+)
  and     b.segment7 = g.segment7(+)
  and     b.latest_opened_year = g.period_year(+)
  and     b.segment1 = p.segment1(+)
  and     b.segment3 = p.segment3(+)
  and     b.segment4 = p.segment4(+)
  and     b.segment6 = p.segment6(+)
  and     b.segment7 = p.segment7(+)
  and     b.latest_opened_year = p.period_year(+)
  and (budget_name = '2015 VMS III K.') -------2015 1/12 SAU E   2015 2/12 VAS E     2015 M VMS PROJ  2015 VMS I K.

 order by b.segment7, b.segment6, b.segment1
