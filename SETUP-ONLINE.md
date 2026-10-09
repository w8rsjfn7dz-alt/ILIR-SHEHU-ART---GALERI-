# ILIR SHEHU ART STUDIO — konfigurimi i albumit online

Faqja aktuale është demonstrim. Fotografitë që shtohen tani ruhen vetëm në shfletuesin ku shtohen. Për një album të përbashkët publik duhet ruajtje online.

## 1. Supabase
- Krijo një projekt te https://supabase.com/dashboard
- Te Authentication > Users krijo përdoruesin e pronarit (email dhe fjalëkalim).
- Te SQL Editor ekzekuto kodin SQL më poshtë.
- Zëvendëso `UUID-I-PRONARIT` me UUID e përdoruesit nga Authentication > Users dhe ekzekuto INSERT-in e fundit.

```sql
create table if not exists public.art_admins (
 user_id uuid primary key references auth.users(id) on delete cascade
);
alter table public.art_admins enable row level security;
create or replace function public.is_art_admin()
returns boolean language sql stable security definer set search_path = ''
as $$ select exists(select 1 from public.art_admins where user_id=(select auth.uid())); $$;
revoke all on function public.is_art_admin() from public;
grant execute on function public.is_art_admin() to anon, authenticated;

create table if not exists public.artworks (
 id uuid primary key default gen_random_uuid(),
 title text not null,
 category text not null check (category in ('painting','drawing','sculpture')),
 description text default '',
 image_url text not null,
 storage_path text not null,
 created_at timestamptz not null default now()
);
alter table public.artworks enable row level security;
create policy "Public gallery read" on public.artworks for select to anon,authenticated using (true);
create policy "Admin upload metadata" on public.artworks for insert to authenticated with check (public.is_art_admin());
create policy "Admin delete metadata" on public.artworks for delete to authenticated using (public.is_art_admin());

insert into storage.buckets (id,name,public,file_size_limit,allowed_mime_types)
values ('artworks','artworks',true,8388608,array['image/jpeg','image/png','image/webp'])
on conflict (id) do nothing;
create policy "Public artwork images" on storage.objects for select to anon,authenticated using (bucket_id='artworks');
create policy "Admin artwork uploads" on storage.objects for insert to authenticated with check (bucket_id='artworks' and public.is_art_admin());
create policy "Admin artwork deletes" on storage.objects for delete to authenticated using (bucket_id='artworks' and public.is_art_admin());

-- Pasi te krijosh perdoruesin:
-- insert into public.art_admins(user_id) values ('UUID-I-PRONARIT');
```

## 2. Çfarë nevojitet për lidhjen me faqen
Duhet **Project URL** dhe **publishable/anon key** nga Supabase. Këto janë të dhëna publike konfigurimi; mos dërgo asnjëherë fjalëkalimin ose çelësin `service_role`.

Pasi të konfigurohet projekti, faqja duhet të kalojë nga ruajtja lokale te Supabase Storage dhe tabela `artworks`. Deri atëherë demonstrimi nuk i publikon fotografitë për vizitorët.

## 3. Siguria
Vetëm llogaria e regjistruar në `art_admins` mund të shtojë ose fshijë fotografi. Vizitorët mund vetëm t'i shohin.
