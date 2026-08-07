-- ============================================================
-- Duck Gift Card — สคริปต์เปลี่ยนประเภทโค้ด (ใช้ซ้ำได้ทุกครั้ง)
-- วิธีใช้: แก้ old_t / new_t สองบรรทัดข้างล่าง แล้วรันทั้งไฟล์
-- เช่น เปลี่ยน Duck 100 → Duck 30:  old_t := '100';  new_t := '30';
-- อย่าลืมแก้ GIFT_TYPES ใน DuckApple.html ให้ตรงกันด้วย
-- รันในโปรเจกต์ Duck (uwgugcnjvoqlmntgxlnv) เท่านั้น
-- ============================================================

do $$
declare
  old_t text := '100';   -- ⬅️ เลขเดิม
  new_t text := '30';    -- ⬅️ เลขใหม่
  r   record;
  src text;
  new_src text;
begin
  if old_t = new_t or new_t !~ '^[0-9]{1,6}$' then
    raise exception 'ค่า old_t/new_t ไม่ถูกต้อง';
  end if;

  -- 1) constraint แบบตัวเลขล้วน (ตั้งครั้งเดียว รองรับทุกเลขในอนาคต ไม่ต้องแก้อีก)
  for r in
    select c.conname, c.conrelid::regclass as tbl
    from pg_constraint c
    where c.contype = 'c'
      and c.connamespace = 'public'::regnamespace
      and pg_get_constraintdef(c.oid) ilike '%gift_type%'
  loop
    execute format('alter table %s drop constraint %I', r.tbl, r.conname);
    execute format(
      'alter table %s add constraint %I check (gift_type ~ ''^[0-9]{1,6}$'')',
      r.tbl, r.conname);
  end loop;

  -- 2) แก้ทุกฟังก์ชันที่อ้างเลขเดิม
  for r in
    select p.oid, p.proname
    from pg_proc p
    join pg_namespace n on n.oid = p.pronamespace
    where n.nspname = 'public' and p.prokind = 'f'
  loop
    src := pg_get_functiondef(r.oid);
    new_src := src;
    -- mapping: like '%OLD%' then 'OLD' → like '%NEW%' then 'NEW'
    new_src := replace(new_src,
      'like ''%' || old_t || '%'' then ''' || old_t || '''',
      'like ''%' || new_t || '%'' then ''' || new_t || '''');
    -- ค่า default (else 'OLD' end)
    new_src := replace(new_src, 'else ''' || old_t || ''' end', 'else ''' || new_t || ''' end');
    -- filter ในรายงาน
    new_src := replace(new_src, 'gift_type = ''' || old_t || '''', 'gift_type = ''' || new_t || '''');
    new_src := replace(new_src, 'gift_type=''' || old_t || '''',   'gift_type=''' || new_t || '''');
    -- ป้ายชื่อและ key ที่ส่งให้หน้าเว็บ
    new_src := replace(new_src, '''Duck ' || old_t || '''', '''Duck ' || new_t || '''');
    new_src := replace(new_src, '''duck' || old_t || '''',  '''duck' || new_t || '''');
    if new_src <> src then
      execute new_src;
      raise notice 'อัปเดตฟังก์ชัน: %', r.proname;
    end if;
  end loop;

  -- 3) ย้ายสต็อกที่ยังไม่ถูกเบิกไปเลขใหม่ (ประวัติการเบิกเดิมไม่แตะ)
  execute format(
    'update codes set gift_type = %L where gift_type = %L and status = ''AVAILABLE''',
    new_t, old_t);
  raise notice 'ย้ายสต็อก % → % เรียบร้อย', old_t, new_t;
end $$;

-- ตรวจสอบ: จำนวนคงเหลือแต่ละประเภท
select gift_type, status, count(*) from codes group by 1, 2 order by 1, 2;
