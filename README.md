This is to inform you that the below procedures have been implemented in a new table EXAD_GWD_DAILY_REMARKS in EPDV:
-- Create new gwd daily_remark record
  PROCEDURE insert_gwd_daily_remark
  (
    p_ep_a_num           IN exad_gwd_daily_remarks.ep_a_num%TYPE,
    p_w_act_dt           IN exad_gwd_daily_remarks.w_act_dt%TYPE,
    p_area               IN exad_gwd_daily_remarks.area%TYPE,
    p_status             IN exad_gwd_daily_remarks.status%TYPE,
    p_w_op_rmk           IN exad_gwd_daily_remarks.w_op_rmk%TYPE,
    p_nxt_24_hr_plan_rmk IN exad_gwd_daily_remarks.nxt_24_hr_plan_rmk%TYPE,
    p_w_drlg_smry_rmk    IN exad_gwd_daily_remarks.w_drlg_smry_rmk%TYPE,
    p_do_commit          IN VARCHAR2 DEFAULT 'Y',
    p_egdr_id            OUT exad_gwd_daily_remarks.egdr_id%TYPE,
    p_return_value       OUT VARCHAR2
  );
  
  -- Update gwd daily_remark record
  PROCEDURE update_gwd_daily_remark
  (
    p_ep_a_num           IN exad_gwd_daily_remarks.ep_a_num%TYPE,
    p_w_act_dt           IN exad_gwd_daily_remarks.w_act_dt%TYPE,
    p_area               IN exad_gwd_daily_remarks.area%TYPE,
    p_status             IN exad_gwd_daily_remarks.status%TYPE,
    p_w_op_rmk           IN exad_gwd_daily_remarks.w_op_rmk%TYPE,
    p_nxt_24_hr_plan_rmk IN exad_gwd_daily_remarks.nxt_24_hr_plan_rmk%TYPE,
    p_w_drlg_smry_rmk    IN exad_gwd_daily_remarks.w_drlg_smry_rmk%TYPE,
    p_egdr_id            IN exad_gwd_daily_remarks.egdr_id%TYPE,
    p_do_commit          IN VARCHAR2 DEFAULT 'Y',
    p_return_value       OUT VARCHAR2
  );
  
  -- Delete gwd daily remark record
  PROCEDURE delete_gwd_daily_remark
  (
    p_egdr_id           IN exad_gwd_daily_remarks.egdr_id%TYPE,
    p_do_commit         IN VARCHAR2 DEFAULT 'Y',
    p_return_value      OUT VARCHAR2
  );

