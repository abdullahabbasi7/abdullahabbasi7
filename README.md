--
-- PostgreSQL database dump
--

\restrict U4sME4TDTrHXSmfG8V6ylb6AJUOYttxl8cvfnG7SEChH1wDdosaBY8kkvK3VaFT

-- Dumped from database version 16.15 (Debian 16.15-1.pgdg13+2)
-- Dumped by pg_dump version 16.15 (Debian 16.15-1.pgdg13+2)

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

--
-- Name: app_current_company(); Type: FUNCTION; Schema: public; Owner: pulse
--

CREATE FUNCTION public.app_current_company() RETURNS uuid
    LANGUAGE sql STABLE
    AS $$
        SELECT NULLIF(current_setting('app.company_id', true), '')::uuid
    $$;


ALTER FUNCTION public.app_current_company() OWNER TO pulse;

--
-- Name: auth_find_user(text, text); Type: FUNCTION; Schema: public; Owner: pulse
--

CREATE FUNCTION public.auth_find_user(p_email text, p_phone text) RETURNS TABLE(id uuid, company_id uuid, full_name text, email text, password_hash text, role text, is_active boolean, locked_until timestamp with time zone)
    LANGUAGE sql STABLE SECURITY DEFINER
    SET search_path TO 'public'
    AS $$
        SELECT u.id, u.company_id, u.full_name, u.email, u.password_hash, u.role, u.is_active,
               u.locked_until
        FROM users u
        WHERE (p_email IS NOT NULL AND lower(u.email) = lower(p_email))
           OR (p_phone IS NOT NULL AND u.phone = p_phone)
        LIMIT 1
    $$;


ALTER FUNCTION public.auth_find_user(p_email text, p_phone text) OWNER TO pulse;

--
-- Name: auth_record_login(uuid, boolean, integer, integer); Type: FUNCTION; Schema: public; Owner: pulse
--

CREATE FUNCTION public.auth_record_login(p_user uuid, p_success boolean, p_max integer, p_lock_minutes integer) RETURNS timestamp with time zone
    LANGUAGE plpgsql SECURITY DEFINER
    SET search_path TO 'public'
    AS $$
    DECLARE
        result timestamptz;
    BEGIN
        IF p_success THEN
            UPDATE users SET failed_logins = 0, locked_until = NULL WHERE id = p_user;
            RETURN NULL;
        END IF;
        UPDATE users
           SET failed_logins = CASE WHEN failed_logins + 1 >= p_max THEN 0 ELSE failed_logins + 1 END,
               locked_until  = CASE WHEN failed_logins + 1 >= p_max
                                    THEN now() + make_interval(mins => p_lock_minutes)
                                    ELSE locked_until END
         WHERE id = p_user
        RETURNING locked_until INTO result;
        RETURN result;
    END
    $$;


ALTER FUNCTION public.auth_record_login(p_user uuid, p_success boolean, p_max integer, p_lock_minutes integer) OWNER TO pulse;

--
-- Name: platform_find_admin(text); Type: FUNCTION; Schema: public; Owner: pulse
--

CREATE FUNCTION public.platform_find_admin(p_email text) RETURNS TABLE(id uuid, email text, full_name text, password_hash text, is_active boolean)
    LANGUAGE sql STABLE SECURITY DEFINER
    SET search_path TO 'public'
    AS $$
        SELECT a.id, a.email, a.full_name, a.password_hash, a.is_active
        FROM platform_admins a WHERE lower(a.email) = lower(p_email)
    $$;


ALTER FUNCTION public.platform_find_admin(p_email text) OWNER TO pulse;

--
-- Name: platform_get_admin(uuid); Type: FUNCTION; Schema: public; Owner: pulse
--

CREATE FUNCTION public.platform_get_admin(p_id uuid) RETURNS TABLE(id uuid, email text, full_name text, is_active boolean)
    LANGUAGE sql STABLE SECURITY DEFINER
    SET search_path TO 'public'
    AS $$
        SELECT a.id, a.email, a.full_name, a.is_active FROM platform_admins a WHERE a.id = p_id
    $$;


ALTER FUNCTION public.platform_get_admin(p_id uuid) OWNER TO pulse;

--
-- Name: platform_list_companies(); Type: FUNCTION; Schema: public; Owner: pulse
--

CREATE FUNCTION public.platform_list_companies() RETURNS TABLE(id uuid, name text, city text, phone text, timezone text, status text, created_at timestamp with time zone, reviewed_at timestamp with time zone, review_note text, reviewed_by_name text, owner_name text, owner_email text, owner_phone text, members bigint, bookings bigint)
    LANGUAGE sql STABLE SECURITY DEFINER
    SET search_path TO 'public'
    AS $$
        SELECT c.id, c.name, c.city, c.phone, c.timezone, c.status, c.created_at, c.reviewed_at,
               c.review_note, a.full_name,
               o.full_name, o.email, o.phone,
               (SELECT count(*) FROM users u WHERE u.company_id = c.id),
               (SELECT count(*) FROM bookings b WHERE b.company_id = c.id)
        FROM companies c
        LEFT JOIN platform_admins a ON a.id = c.reviewed_by
        LEFT JOIN LATERAL (
            SELECT u.full_name, u.email, u.phone FROM users u
            WHERE u.company_id = c.id AND u.role = 'owner' ORDER BY u.created_at LIMIT 1
        ) o ON true
        ORDER BY c.created_at DESC
    $$;


ALTER FUNCTION public.platform_list_companies() OWNER TO pulse;

--
-- Name: platform_set_company_status(uuid, text, text, uuid); Type: FUNCTION; Schema: public; Owner: pulse
--

CREATE FUNCTION public.platform_set_company_status(p_company uuid, p_status text, p_note text, p_admin uuid) RETURNS text
    LANGUAGE plpgsql SECURITY DEFINER
    SET search_path TO 'public'
    AS $$
    DECLARE
        result text;
    BEGIN
        IF p_status NOT IN ('active', 'rejected', 'suspended') THEN
            RAISE EXCEPTION 'invalid status %', p_status;
        END IF;
        UPDATE companies
           SET status = p_status, review_note = p_note, reviewed_at = now(), reviewed_by = p_admin
         WHERE id = p_company
        RETURNING status INTO result;
        RETURN result;
    END
    $$;


ALTER FUNCTION public.platform_set_company_status(p_company uuid, p_status text, p_note text, p_admin uuid) OWNER TO pulse;

SET default_tablespace = '';

SET default_table_access_method = heap;

--
-- Name: alembic_version; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.alembic_version (
    version_num character varying(32) NOT NULL
);


ALTER TABLE public.alembic_version OWNER TO pulse;

--
-- Name: attendance_logs; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.attendance_logs (
    id uuid NOT NULL,
    company_id uuid NOT NULL,
    staff_id uuid NOT NULL,
    kind text NOT NULL,
    logged_at timestamp with time zone DEFAULT now() NOT NULL,
    booking_id uuid,
    site text,
    method text NOT NULL,
    match_score real,
    liveness_score real,
    note text,
    recorded_by uuid,
    created_at timestamp with time zone DEFAULT now() NOT NULL,
    CONSTRAINT attendance_logs_check CHECK (((method = 'face'::text) OR (note IS NOT NULL))),
    CONSTRAINT attendance_logs_kind_check CHECK ((kind = ANY (ARRAY['in'::text, 'out'::text]))),
    CONSTRAINT attendance_logs_method_check CHECK ((method = ANY (ARRAY['face'::text, 'manual'::text])))
);


ALTER TABLE public.attendance_logs OWNER TO pulse;

--
-- Name: booking_history; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.booking_history (
    id uuid NOT NULL,
    company_id uuid NOT NULL,
    booking_id uuid NOT NULL,
    actor_id uuid,
    action text NOT NULL,
    detail text,
    created_at timestamp with time zone DEFAULT now() NOT NULL
);


ALTER TABLE public.booking_history OWNER TO pulse;

--
-- Name: bookings; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.bookings (
    id uuid NOT NULL,
    company_id uuid NOT NULL,
    client_id uuid NOT NULL,
    event_type text NOT NULL,
    event_date date NOT NULL,
    start_time time without time zone,
    end_time time without time zone,
    venue text NOT NULL,
    guest_count integer NOT NULL,
    menu text,
    package text,
    add_ons text,
    quoted_amount numeric(14,2),
    status text DEFAULT 'inquiry'::text NOT NULL,
    progress text DEFAULT 'scheduled'::text NOT NULL,
    inquiry_text text,
    notes text,
    created_by uuid,
    created_at timestamp with time zone DEFAULT now() NOT NULL,
    updated_at timestamp with time zone DEFAULT now() NOT NULL,
    CONSTRAINT bookings_check CHECK (((end_time IS NULL) OR (start_time IS NULL) OR (end_time > start_time))),
    CONSTRAINT bookings_event_type_check CHECK ((event_type = ANY (ARRAY['wedding'::text, 'mehndi'::text, 'walima'::text, 'engagement'::text, 'birthday'::text, 'corporate'::text, 'conference'::text, 'religious'::text, 'other'::text]))),
    CONSTRAINT bookings_guest_count_check CHECK (((guest_count > 0) AND (guest_count <= 100000))),
    CONSTRAINT bookings_progress_check CHECK ((progress = ANY (ARRAY['scheduled'::text, 'dispatched'::text, 'setup'::text, 'serving'::text, 'wind_up'::text, 'done'::text]))),
    CONSTRAINT bookings_quoted_amount_check CHECK (((quoted_amount IS NULL) OR (quoted_amount >= (0)::numeric))),
    CONSTRAINT bookings_status_check CHECK ((status = ANY (ARRAY['inquiry'::text, 'quoted'::text, 'confirmed'::text, 'in_progress'::text, 'completed'::text, 'cancelled'::text])))
);


ALTER TABLE public.bookings OWNER TO pulse;

--
-- Name: clients; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.clients (
    id uuid NOT NULL,
    company_id uuid NOT NULL,
    name text NOT NULL,
    phone text,
    email text,
    address text,
    notes text,
    created_at timestamp with time zone DEFAULT now() NOT NULL,
    CONSTRAINT clients_name_check CHECK ((length(TRIM(BOTH FROM name)) > 0))
);


ALTER TABLE public.clients OWNER TO pulse;

--
-- Name: companies; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.companies (
    id uuid NOT NULL,
    name text NOT NULL,
    phone text,
    address text,
    city text,
    logo_url text,
    created_at timestamp with time zone DEFAULT now() NOT NULL,
    timezone text DEFAULT 'Asia/Karachi'::text NOT NULL,
    status text DEFAULT 'pending'::text NOT NULL,
    reviewed_at timestamp with time zone,
    reviewed_by uuid,
    review_note text,
    CONSTRAINT companies_name_check CHECK ((length(TRIM(BOTH FROM name)) > 0)),
    CONSTRAINT companies_status_check CHECK ((status = ANY (ARRAY['pending'::text, 'active'::text, 'rejected'::text, 'suspended'::text])))
);


ALTER TABLE public.companies OWNER TO pulse;

--
-- Name: face_embeddings; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.face_embeddings (
    id uuid NOT NULL,
    company_id uuid NOT NULL,
    staff_id uuid NOT NULL,
    embedding bytea NOT NULL,
    quality real,
    created_at timestamp with time zone DEFAULT now() NOT NULL
);


ALTER TABLE public.face_embeddings OWNER TO pulse;

--
-- Name: inventory_items; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.inventory_items (
    id uuid NOT NULL,
    company_id uuid NOT NULL,
    name text NOT NULL,
    category text NOT NULL,
    unit text DEFAULT 'pcs'::text NOT NULL,
    total_quantity integer NOT NULL,
    min_quantity integer DEFAULT 0 NOT NULL,
    condition_note text,
    qr_token text NOT NULL,
    is_active boolean DEFAULT true NOT NULL,
    created_at timestamp with time zone DEFAULT now() NOT NULL,
    CONSTRAINT inventory_items_category_check CHECK ((category = ANY (ARRAY['dinner_plate'::text, 'dessert_plate'::text, 'bowl'::text, 'glass'::text, 'cup'::text, 'cutlery_set'::text, 'chafing_dish'::text, 'serving_dish'::text, 'chair'::text, 'table'::text, 'tablecloth'::text, 'other'::text]))),
    CONSTRAINT inventory_items_min_quantity_check CHECK ((min_quantity >= 0)),
    CONSTRAINT inventory_items_name_check CHECK ((length(TRIM(BOTH FROM name)) > 0)),
    CONSTRAINT inventory_items_total_quantity_check CHECK ((total_quantity >= 0))
);


ALTER TABLE public.inventory_items OWNER TO pulse;

--
-- Name: item_movements; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.item_movements (
    id uuid NOT NULL,
    company_id uuid NOT NULL,
    item_id uuid NOT NULL,
    booking_id uuid NOT NULL,
    direction text NOT NULL,
    quantity integer NOT NULL,
    damaged integer DEFAULT 0 NOT NULL,
    note text,
    recorded_by uuid,
    created_at timestamp with time zone DEFAULT now() NOT NULL,
    CONSTRAINT item_movements_check CHECK (((quantity + damaged) > 0)),
    CONSTRAINT item_movements_check1 CHECK (((direction = 'in'::text) OR (damaged = 0))),
    CONSTRAINT item_movements_damaged_check CHECK ((damaged >= 0)),
    CONSTRAINT item_movements_direction_check CHECK ((direction = ANY (ARRAY['out'::text, 'in'::text, 'lost'::text]))),
    CONSTRAINT item_movements_quantity_check CHECK ((quantity >= 0))
);


ALTER TABLE public.item_movements OWNER TO pulse;

--
-- Name: platform_admins; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.platform_admins (
    id uuid NOT NULL,
    email text NOT NULL,
    full_name text NOT NULL,
    password_hash text NOT NULL,
    is_active boolean DEFAULT true NOT NULL,
    created_at timestamp with time zone DEFAULT now() NOT NULL
);


ALTER TABLE public.platform_admins OWNER TO pulse;

--
-- Name: staff_members; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.staff_members (
    id uuid NOT NULL,
    company_id uuid NOT NULL,
    full_name text NOT NULL,
    designation text DEFAULT 'Staff'::text NOT NULL,
    phone text,
    daily_wage numeric(12,2) DEFAULT 0 NOT NULL,
    user_id uuid,
    face_consent boolean DEFAULT false NOT NULL,
    face_enrolled_at timestamp with time zone,
    is_active boolean DEFAULT true NOT NULL,
    created_at timestamp with time zone DEFAULT now() NOT NULL,
    CONSTRAINT staff_members_daily_wage_check CHECK ((daily_wage >= (0)::numeric)),
    CONSTRAINT staff_members_full_name_check CHECK ((length(TRIM(BOTH FROM full_name)) > 0))
);


ALTER TABLE public.staff_members OWNER TO pulse;

--
-- Name: users; Type: TABLE; Schema: public; Owner: pulse
--

CREATE TABLE public.users (
    id uuid NOT NULL,
    company_id uuid NOT NULL,
    full_name text NOT NULL,
    email text NOT NULL,
    password_hash text NOT NULL,
    role text NOT NULL,
    is_active boolean DEFAULT true NOT NULL,
    created_at timestamp with time zone DEFAULT now() NOT NULL,
    phone text,
    failed_logins integer DEFAULT 0 NOT NULL,
    locked_until timestamp with time zone,
    password_changed_at timestamp with time zone DEFAULT now() NOT NULL,
    avatar text,
    job_title text,
    CONSTRAINT users_avatar_check CHECK (((avatar IS NULL) OR (length(avatar) <= 120000))),
    CONSTRAINT users_job_title_check CHECK (((job_title IS NULL) OR (length(job_title) <= 80))),
    CONSTRAINT users_phone_format CHECK (((phone IS NULL) OR (phone ~ '^\+[0-9]{8,15}$'::text))),
    CONSTRAINT users_role_check CHECK ((role = ANY (ARRAY['owner'::text, 'manager'::text, 'staff'::text, 'driver'::text, 'accounts'::text])))
);


ALTER TABLE public.users OWNER TO pulse;

--
-- Data for Name: alembic_version; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.alembic_version (version_num) FROM stdin;
0008_user_profiles
\.


--
-- Data for Name: attendance_logs; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.attendance_logs (id, company_id, staff_id, kind, logged_at, booking_id, site, method, match_score, liveness_score, note, recorded_by, created_at) FROM stdin;
\.


--
-- Data for Name: booking_history; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.booking_history (id, company_id, booking_id, actor_id, action, detail, created_at) FROM stdin;
50d8b6ca-a5ae-4e94-b2af-4bac7a5e998d	4d345103-3a71-4789-a36f-3787d227cf76	e3ece8b9-4e38-4e8a-97fa-cd1bb3b7d358	84594b06-a269-4d82-87bc-78e71ac7fa00	created	Status: in_progress	2026-11-08 21:26:41.286579+00
3711871e-8a7d-47e7-9343-bebff33b8d1a	4d345103-3a71-4789-a36f-3787d227cf76	e6d058b2-3d2c-4be5-8613-2ed3927ac896	84594b06-a269-4d82-87bc-78e71ac7fa00	created	Status: confirmed	2026-11-08 21:26:41.286579+00
c1694ad7-d307-49c2-b213-4e76ecc42c89	4d345103-3a71-4789-a36f-3787d227cf76	23668866-1f89-4595-9db3-b3ddb0a13580	84594b06-a269-4d82-87bc-78e71ac7fa00	created	Status: confirmed	2026-11-08 21:26:41.286579+00
36a8c0c6-cfd3-4d7a-944c-aad967a91857	4d345103-3a71-4789-a36f-3787d227cf76	0dd121ef-589f-4b57-85b3-8716b54d33fa	84594b06-a269-4d82-87bc-78e71ac7fa00	created	Status: quoted	2026-11-08 21:26:41.286579+00
a3023b4a-7f9e-4edd-bbf2-2ccfa4e22fb1	4d345103-3a71-4789-a36f-3787d227cf76	6c12e61c-819a-4aa3-b898-0cb290e38bb1	84594b06-a269-4d82-87bc-78e71ac7fa00	created	Status: inquiry	2026-11-08 21:26:41.286579+00
25d3eda4-adb7-4b2d-8b44-e135940ca61a	4d345103-3a71-4789-a36f-3787d227cf76	00148b27-b2b1-45e2-be88-b452b55708ee	84594b06-a269-4d82-87bc-78e71ac7fa00	created	Status: completed	2026-11-08 21:26:41.286579+00
\.


--
-- Data for Name: bookings; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.bookings (id, company_id, client_id, event_type, event_date, start_time, end_time, venue, guest_count, menu, package, add_ons, quoted_amount, status, progress, inquiry_text, notes, created_by, created_at, updated_at) FROM stdin;
e3ece8b9-4e38-4e8a-97fa-cd1bb3b7d358	4d345103-3a71-4789-a36f-3787d227cf76	2a5f4eca-e9f5-455f-9bd5-219b5ef3b9c5	wedding	2026-11-09	19:00:00	23:30:00	Grand Marquee, DHA Phase 2	450	Chicken biryani, mutton qorma, seekh kabab, naan, raita, kheer, soft drinks	\N	\N	\N	in_progress	serving	\N	\N	84594b06-a269-4d82-87bc-78e71ac7fa00	2026-11-08 21:26:41.286579+00	2026-11-08 21:26:41.286579+00
e6d058b2-3d2c-4be5-8613-2ed3927ac896	4d345103-3a71-4789-a36f-3787d227cf76	efb8760a-9601-41eb-bc07-822fe570dc57	corporate	2026-11-09	13:00:00	15:30:00	Tech Valley office, Blue Area	120	Chicken tikka, pasta, salad, brownies, green tea	\N	\N	\N	confirmed	scheduled	\N	\N	84594b06-a269-4d82-87bc-78e71ac7fa00	2026-11-08 21:26:41.286579+00	2026-11-08 21:26:41.286579+00
23668866-1f89-4595-9db3-b3ddb0a13580	4d345103-3a71-4789-a36f-3787d227cf76	973e97d1-a1de-4b43-9c75-6f5fced65fa4	walima	2026-11-11	20:00:00	23:00:00	Emerald Banquet Hall, F-7	600	Mutton qorma, chicken biryani, salad, naan, firni	\N	\N	\N	confirmed	scheduled	\N	\N	84594b06-a269-4d82-87bc-78e71ac7fa00	2026-11-08 21:26:41.286579+00	2026-11-08 21:26:41.286579+00
0dd121ef-589f-4b57-85b3-8716b54d33fa	4d345103-3a71-4789-a36f-3787d227cf76	4581944b-3489-4d5b-a5b5-0fa351d89e96	mehndi	2026-11-14	20:30:00	\N	Siddiqui residence, E-11	150	Chicken karahi, naan, gajar ka halwa, kashmiri chai	\N	\N	\N	quoted	scheduled	\N	\N	84594b06-a269-4d82-87bc-78e71ac7fa00	2026-11-08 21:26:41.286579+00	2026-11-08 21:26:41.286579+00
6c12e61c-819a-4aa3-b898-0cb290e38bb1	4d345103-3a71-4789-a36f-3787d227cf76	030635b0-3613-4a3e-81e2-008529719cb4	birthday	2026-11-18	17:00:00	20:00:00	Lakeview Lawn, Bani Gala	80	Sandwiches, cake, ice cream, soft drinks	\N	\N	\N	inquiry	scheduled	\N	\N	84594b06-a269-4d82-87bc-78e71ac7fa00	2026-11-08 21:26:41.286579+00	2026-11-08 21:26:41.286579+00
00148b27-b2b1-45e2-be88-b452b55708ee	4d345103-3a71-4789-a36f-3787d227cf76	973e97d1-a1de-4b43-9c75-6f5fced65fa4	engagement	2026-11-06	19:00:00	22:00:00	Pearl Hall, G-11	300	Chicken biryani, chicken qorma, naan, custard	\N	\N	\N	completed	done	\N	\N	84594b06-a269-4d82-87bc-78e71ac7fa00	2026-11-08 21:26:41.286579+00	2026-11-08 21:26:41.286579+00
\.


--
-- Data for Name: clients; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.clients (id, company_id, name, phone, email, address, notes, created_at) FROM stdin;
2a5f4eca-e9f5-455f-9bd5-219b5ef3b9c5	4d345103-3a71-4789-a36f-3787d227cf76	Sara Ahmed	+923211234567	\N	\N	\N	2026-11-08 21:26:41.286579+00
973e97d1-a1de-4b43-9c75-6f5fced65fa4	4d345103-3a71-4789-a36f-3787d227cf76	Bilal Chaudhry	+923331112233	\N	\N	\N	2026-11-08 21:26:41.286579+00
4581944b-3489-4d5b-a5b5-0fa351d89e96	4d345103-3a71-4789-a36f-3787d227cf76	Ayesha Siddiqui	+923004445566	\N	\N	\N	2026-11-08 21:26:41.286579+00
efb8760a-9601-41eb-bc07-822fe570dc57	4d345103-3a71-4789-a36f-3787d227cf76	Tech Valley Pvt Ltd	+92512345678	\N	\N	\N	2026-11-08 21:26:41.286579+00
030635b0-3613-4a3e-81e2-008529719cb4	4d345103-3a71-4789-a36f-3787d227cf76	Usman Tariq	+923459876543	\N	\N	\N	2026-11-08 21:26:41.286579+00
\.


--
-- Data for Name: companies; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.companies (id, name, phone, address, city, logo_url, created_at, timezone, status, reviewed_at, reviewed_by, review_note) FROM stdin;
4d345103-3a71-4789-a36f-3787d227cf76	Royal Caterers	+923000000001	\N	Islamabad	\N	2026-11-08 21:26:41.286579+00	Asia/Karachi	active	2026-11-08 21:26:41.737879+00	\N	Demo company
a76c7345-ad2d-4392-a745-f95c8743fdae	Crescent Events	+971500000001	\N	Dubai	\N	2026-11-08 21:26:41.741273+00	Asia/Dubai	active	2026-11-08 21:26:41.866674+00	\N	Demo company
\.


--
-- Data for Name: face_embeddings; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.face_embeddings (id, company_id, staff_id, embedding, quality, created_at) FROM stdin;
\.


--
-- Data for Name: inventory_items; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.inventory_items (id, company_id, name, category, unit, total_quantity, min_quantity, condition_note, qr_token, is_active, created_at) FROM stdin;
95e3af40-32db-4ea0-92cb-347b83305912	4d345103-3a71-4789-a36f-3787d227cf76	Quarter plate	dessert_plate	pcs	800	200	\N	d0867087b125	t	2026-11-08 21:26:41.286579+00
80b72113-22ac-4126-ad93-4fd98e971ffe	4d345103-3a71-4789-a36f-3787d227cf76	Dessert bowl	bowl	pcs	900	200	\N	f50becceb9e3	t	2026-11-08 21:26:41.286579+00
8d32d996-1911-483b-90cc-ec8173efc140	4d345103-3a71-4789-a36f-3787d227cf76	Tea cup with saucer	cup	pcs	600	150	\N	6173a48befda	t	2026-11-08 21:26:41.286579+00
08929398-04f2-40f0-8e19-2582149a7c7f	4d345103-3a71-4789-a36f-3787d227cf76	Cutlery set (spoon, fork, knife)	cutlery_set	pcs	1300	300	\N	c8af8b49617f	t	2026-11-08 21:26:41.286579+00
6a03fd3a-26c1-498f-b9c7-fee65f2bd933	4d345103-3a71-4789-a36f-3787d227cf76	Chafing dish, steel	chafing_dish	pcs	40	10	\N	9c007c348ae6	t	2026-11-08 21:26:41.286579+00
fdbd96d9-b660-4893-abeb-4451697154fb	4d345103-3a71-4789-a36f-3787d227cf76	Serving platter	serving_dish	pcs	120	30	\N	2de03e9030e8	t	2026-11-08 21:26:41.286579+00
f3ce131b-ba6a-4ed2-a7e6-5fe491f78cf6	4d345103-3a71-4789-a36f-3787d227cf76	Banquet chair	chair	pcs	1000	250	\N	20ba7aa02e08	t	2026-11-08 21:26:41.286579+00
90703c62-b7b0-496d-99cf-4aff185f799b	4d345103-3a71-4789-a36f-3787d227cf76	Round table (8 seats)	table	pcs	120	30	\N	3e8e6a9b15f2	t	2026-11-08 21:26:41.286579+00
b5e1ac61-3afb-44f7-bd6f-b96f2360fcab	4d345103-3a71-4789-a36f-3787d227cf76	Tablecloth, white	tablecloth	pcs	140	40	\N	9edb9f69797b	t	2026-11-08 21:26:41.286579+00
4cb58840-eddc-4110-8fb5-e9dfd11c6e4d	4d345103-3a71-4789-a36f-3787d227cf76	Dinner plate, white	dinner_plate	pcs	1194	300	\N	74d3805a539e	t	2026-11-08 21:26:41.286579+00
f82cbd53-bcba-4902-9ed3-dba1a7a2a54e	4d345103-3a71-4789-a36f-3787d227cf76	Water glass	glass	pcs	1491	400	\N	13c7b7299012	t	2026-11-08 21:26:41.286579+00
\.


--
-- Data for Name: item_movements; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.item_movements (id, company_id, item_id, booking_id, direction, quantity, damaged, note, recorded_by, created_at) FROM stdin;
8658351f-7e6e-47b0-8114-9cc179a9393c	4d345103-3a71-4789-a36f-3787d227cf76	4cb58840-eddc-4110-8fb5-e9dfd11c6e4d	e3ece8b9-4e38-4e8a-97fa-cd1bb3b7d358	out	620	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
0773e805-5e97-4751-ba74-bc4301590762	4d345103-3a71-4789-a36f-3787d227cf76	80b72113-22ac-4126-ad93-4fd98e971ffe	e3ece8b9-4e38-4e8a-97fa-cd1bb3b7d358	out	520	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
3d678f9f-f3cb-4a01-b626-54944af1fa99	4d345103-3a71-4789-a36f-3787d227cf76	f82cbd53-bcba-4902-9ed3-dba1a7a2a54e	e3ece8b9-4e38-4e8a-97fa-cd1bb3b7d358	out	800	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
0e32ca16-d8ad-4c0d-9437-80db3cbd4930	4d345103-3a71-4789-a36f-3787d227cf76	08929398-04f2-40f0-8e19-2582149a7c7f	e3ece8b9-4e38-4e8a-97fa-cd1bb3b7d358	out	600	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
2ff438ee-8b58-471c-88f3-d757ff6985f3	4d345103-3a71-4789-a36f-3787d227cf76	6a03fd3a-26c1-498f-b9c7-fee65f2bd933	e3ece8b9-4e38-4e8a-97fa-cd1bb3b7d358	out	14	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
ebcd18ea-3eb0-47e4-9b6b-45dd7555573e	4d345103-3a71-4789-a36f-3787d227cf76	f3ce131b-ba6a-4ed2-a7e6-5fe491f78cf6	e3ece8b9-4e38-4e8a-97fa-cd1bb3b7d358	out	430	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
25dbb0d8-7493-4077-9ae5-a9aa5f7b7c7f	4d345103-3a71-4789-a36f-3787d227cf76	90703c62-b7b0-496d-99cf-4aff185f799b	e3ece8b9-4e38-4e8a-97fa-cd1bb3b7d358	out	56	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
da1b70ac-16e3-470b-b175-c00245af70da	4d345103-3a71-4789-a36f-3787d227cf76	4cb58840-eddc-4110-8fb5-e9dfd11c6e4d	00148b27-b2b1-45e2-be88-b452b55708ee	out	400	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
74b077d9-9a60-496c-9cb5-94187d4cb709	4d345103-3a71-4789-a36f-3787d227cf76	4cb58840-eddc-4110-8fb5-e9dfd11c6e4d	00148b27-b2b1-45e2-be88-b452b55708ee	in	372	6	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
5e9ba8cd-5beb-4122-af72-cb027119ca0f	4d345103-3a71-4789-a36f-3787d227cf76	f82cbd53-bcba-4902-9ed3-dba1a7a2a54e	00148b27-b2b1-45e2-be88-b452b55708ee	out	450	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
055797f7-e12c-43b2-9ff1-2eace9f94f3f	4d345103-3a71-4789-a36f-3787d227cf76	f82cbd53-bcba-4902-9ed3-dba1a7a2a54e	00148b27-b2b1-45e2-be88-b452b55708ee	in	431	9	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
f65ebf17-9437-4f35-b730-65f10f9d9b66	4d345103-3a71-4789-a36f-3787d227cf76	f3ce131b-ba6a-4ed2-a7e6-5fe491f78cf6	00148b27-b2b1-45e2-be88-b452b55708ee	out	290	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
33c6f3c0-2aef-4842-9000-a3dcf134fc2e	4d345103-3a71-4789-a36f-3787d227cf76	f3ce131b-ba6a-4ed2-a7e6-5fe491f78cf6	00148b27-b2b1-45e2-be88-b452b55708ee	in	290	0	\N	cffc6951-2792-45fa-97e6-3ae2d2d8155e	2026-11-08 21:26:41.286579+00
\.


--
-- Data for Name: platform_admins; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.platform_admins (id, email, full_name, password_hash, is_active, created_at) FROM stdin;
c069139f-0cf0-44b2-87d3-bde0b14e16ab	admin@platform-demo.dev	Platform Admin	scrypt$16384$8$1$SA-HnIrgCsnqggro5eYPnQ==$yttO_y-gCX3TcqGCcyqSvdj5H6lbxGfsn8OUoD3zaE56ipv4p7IapLWrPqiQutLIa9x48NmfBFa9Vq5wvOuKXw==	t	2026-11-08 21:26:41.885068+00
\.


--
-- Data for Name: staff_members; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.staff_members (id, company_id, full_name, designation, phone, daily_wage, user_id, face_consent, face_enrolled_at, is_active, created_at) FROM stdin;
7950eca5-f2ed-4689-be8b-f8ace5e2553d	4d345103-3a71-4789-a36f-3787d227cf76	Imran Khan	Head waiter	\N	3000.00	\N	t	\N	t	2026-11-08 21:26:41.286579+00
ebae2366-522b-4c24-b95f-798115d03a18	4d345103-3a71-4789-a36f-3787d227cf76	Zeeshan Akhtar	Waiter	\N	2200.00	\N	t	\N	t	2026-11-08 21:26:41.286579+00
94f58f14-68f3-40f8-9525-bcab02086446	4d345103-3a71-4789-a36f-3787d227cf76	Faisal Mehmood	Waiter	\N	2200.00	\N	t	\N	t	2026-11-08 21:26:41.286579+00
4f7810e8-b3b7-4a52-922f-e577b8ebc9fd	4d345103-3a71-4789-a36f-3787d227cf76	Rashid Iqbal	Cook	\N	3500.00	\N	t	\N	t	2026-11-08 21:26:41.286579+00
8d306957-0021-4208-9d9a-7a07015b0bb0	4d345103-3a71-4789-a36f-3787d227cf76	Shazia Parveen	Decorator	\N	2800.00	\N	t	\N	t	2026-11-08 21:26:41.286579+00
75dfd68d-989c-4c15-851c-ba548ac984ba	4d345103-3a71-4789-a36f-3787d227cf76	Kashif Ali	Driver	\N	2500.00	\N	t	\N	t	2026-11-08 21:26:41.286579+00
c68f6ab2-252d-4746-9e2e-171c7e917a94	4d345103-3a71-4789-a36f-3787d227cf76	abc	Waiter	+923005234713	10000.00	\N	t	\N	t	2026-11-08 21:45:19.488774+00
\.


--
-- Data for Name: users; Type: TABLE DATA; Schema: public; Owner: pulse
--

COPY public.users (id, company_id, full_name, email, password_hash, role, is_active, created_at, phone, failed_logins, locked_until, password_changed_at, avatar, job_title) FROM stdin;
84594b06-a269-4d82-87bc-78e71ac7fa00	4d345103-3a71-4789-a36f-3787d227cf76	Sana Malik	manager@royal-demo.pk	scrypt$16384$8$1$wgofXmz6LxIn4x0ARJG1oQ==$5J1EPtp36E6YtDFBMESGmZMRHjoIT51rPeehzZewnQ98YubwNODhVYrX08GeknWKiry9i4J3c-y5je-DlvIn0g==	manager	t	2026-11-08 21:26:41.286579+00	+923000000002	0	\N	2026-11-08 21:26:41.286579+00	\N	\N
cffc6951-2792-45fa-97e6-3ae2d2d8155e	4d345103-3a71-4789-a36f-3787d227cf76	Imran Khan	staff@royal-demo.pk	scrypt$16384$8$1$YD_jIjmFMYoNmDHS9-Bgyw==$-RcAEI13TbfIfTSnV1dLYZg4_M7-4tejNK-cdko3LVJkeSMNPAqVkJi1ozhq__8JbnooBZu7xQoKrffdkLAhhA==	staff	t	2026-11-08 21:26:41.286579+00	+923000000003	0	\N	2026-11-08 21:26:41.286579+00	\N	\N
d477ead8-20a9-4358-a640-7a21a686afbf	4d345103-3a71-4789-a36f-3787d227cf76	Kashif Ali	driver@royal-demo.pk	scrypt$16384$8$1$o8GrUctKAJ45WSOrJpk_8g==$xrV1mQJ_oUjPmCMghO-SWMJPgvpTdsm5u5rK7Y67tLc1GAM3cquwTDAz36HU_xpm5IejHnDua155y3TZF7aInA==	driver	t	2026-11-08 21:26:41.286579+00	+923000000004	0	\N	2026-11-08 21:26:41.286579+00	\N	\N
e1c43ff2-9594-4ebf-acab-0489aa84bc2f	4d345103-3a71-4789-a36f-3787d227cf76	Nadia Hussain	accounts@royal-demo.pk	scrypt$16384$8$1$boSOXMHf2rkM54ljr8VXmA==$NUsOA1rViokPpIlEEYMEhBspCtqz0KevCHlGUJkHDxHg8VCjspVVuCNLBqTWa5icS980tEXokMp2iyJdQIM7Mw==	accounts	t	2026-11-08 21:26:41.286579+00	+923000000005	0	\N	2026-11-08 21:26:41.286579+00	\N	\N
1fec3340-91e9-464b-8ebc-53984dc889f9	a76c7345-ad2d-4392-a745-f95c8743fdae	Omar Siddiqui	owner@crescent-demo.ae	scrypt$16384$8$1$Dz20Z5ec6RwTXZJ7_OU48A==$fjBH08_kc_EwrSWDHdmhuQbtxkMGTtlv1YB-dku5fsPx0OTy3sJbFCm8CZi6flISUPs_GEQjK1KZyfYv7vBvoQ==	owner	t	2026-11-08 21:26:41.741273+00	+971500000001	0	\N	2026-11-08 21:26:41.741273+00	\N	\N
106ebd67-11c5-4ade-aa0a-4b5ff299a51c	a76c7345-ad2d-4392-a745-f95c8743fdae	Layla Haddad	manager@crescent-demo.ae	scrypt$16384$8$1$j-v-eiZg5JQTixJH6ETUBA==$1w2cGZHvWhi_qQad0TOUfU9s-N8iWZzQstixNjued2N9wIiJbKqzhIo8S2cnaQ9Y5fbOfPRYRez47OAl5B-mTA==	manager	t	2026-11-08 21:26:41.741273+00	+971500000002	0	\N	2026-11-08 21:26:41.741273+00	\N	\N
6d9d6af8-6947-401a-8c1a-ed9215574c6e	4d345103-3a71-4789-a36f-3787d227cf76	Hamid Raza	owner@royal-demo.pk	scrypt$16384$8$1$ePdqJFnE7lpyYjwJxzrFew==$ULcGwDILJE1aXSS13JH9gR17hfK3YeezKahQewrmVR9jrto7wQtXIK046NOukve5WcL1pdsosrT0-G6a6CQZWg==	owner	t	2026-11-08 21:26:41.286579+00	+923000000001	0	\N	2026-11-08 21:26:41.286579+00	\N	\N
\.


--
-- Name: alembic_version alembic_version_pkc; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.alembic_version
    ADD CONSTRAINT alembic_version_pkc PRIMARY KEY (version_num);


--
-- Name: attendance_logs attendance_logs_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.attendance_logs
    ADD CONSTRAINT attendance_logs_pkey PRIMARY KEY (id);


--
-- Name: booking_history booking_history_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.booking_history
    ADD CONSTRAINT booking_history_pkey PRIMARY KEY (id);


--
-- Name: bookings bookings_id_company_id_key; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.bookings
    ADD CONSTRAINT bookings_id_company_id_key UNIQUE (id, company_id);


--
-- Name: bookings bookings_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.bookings
    ADD CONSTRAINT bookings_pkey PRIMARY KEY (id);


--
-- Name: clients clients_id_company_id_key; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.clients
    ADD CONSTRAINT clients_id_company_id_key UNIQUE (id, company_id);


--
-- Name: clients clients_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.clients
    ADD CONSTRAINT clients_pkey PRIMARY KEY (id);


--
-- Name: companies companies_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.companies
    ADD CONSTRAINT companies_pkey PRIMARY KEY (id);


--
-- Name: face_embeddings face_embeddings_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.face_embeddings
    ADD CONSTRAINT face_embeddings_pkey PRIMARY KEY (id);


--
-- Name: inventory_items inventory_items_id_company_id_key; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.inventory_items
    ADD CONSTRAINT inventory_items_id_company_id_key UNIQUE (id, company_id);


--
-- Name: inventory_items inventory_items_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.inventory_items
    ADD CONSTRAINT inventory_items_pkey PRIMARY KEY (id);


--
-- Name: inventory_items inventory_items_qr_token_key; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.inventory_items
    ADD CONSTRAINT inventory_items_qr_token_key UNIQUE (qr_token);


--
-- Name: item_movements item_movements_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.item_movements
    ADD CONSTRAINT item_movements_pkey PRIMARY KEY (id);


--
-- Name: platform_admins platform_admins_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.platform_admins
    ADD CONSTRAINT platform_admins_pkey PRIMARY KEY (id);


--
-- Name: staff_members staff_members_id_company_id_key; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.staff_members
    ADD CONSTRAINT staff_members_id_company_id_key UNIQUE (id, company_id);


--
-- Name: staff_members staff_members_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.staff_members
    ADD CONSTRAINT staff_members_pkey PRIMARY KEY (id);


--
-- Name: users users_id_company_key; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.users
    ADD CONSTRAINT users_id_company_key UNIQUE (id, company_id);


--
-- Name: users users_pkey; Type: CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.users
    ADD CONSTRAINT users_pkey PRIMARY KEY (id);


--
-- Name: attendance_logs_company_id_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX attendance_logs_company_id_idx ON public.attendance_logs USING btree (company_id);


--
-- Name: attendance_logs_staff_time_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX attendance_logs_staff_time_idx ON public.attendance_logs USING btree (staff_id, logged_at);


--
-- Name: booking_history_booking_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX booking_history_booking_idx ON public.booking_history USING btree (booking_id);


--
-- Name: booking_history_company_id_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX booking_history_company_id_idx ON public.booking_history USING btree (company_id);


--
-- Name: bookings_company_id_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX bookings_company_id_idx ON public.bookings USING btree (company_id);


--
-- Name: bookings_date_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX bookings_date_idx ON public.bookings USING btree (company_id, event_date);


--
-- Name: clients_company_id_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX clients_company_id_idx ON public.clients USING btree (company_id);


--
-- Name: face_embeddings_company_id_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX face_embeddings_company_id_idx ON public.face_embeddings USING btree (company_id);


--
-- Name: face_embeddings_staff_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX face_embeddings_staff_idx ON public.face_embeddings USING btree (staff_id);


--
-- Name: inventory_items_company_id_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX inventory_items_company_id_idx ON public.inventory_items USING btree (company_id);


--
-- Name: item_movements_booking_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX item_movements_booking_idx ON public.item_movements USING btree (booking_id);


--
-- Name: item_movements_company_id_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX item_movements_company_id_idx ON public.item_movements USING btree (company_id);


--
-- Name: item_movements_item_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX item_movements_item_idx ON public.item_movements USING btree (item_id);


--
-- Name: platform_admins_email_unique; Type: INDEX; Schema: public; Owner: pulse
--

CREATE UNIQUE INDEX platform_admins_email_unique ON public.platform_admins USING btree (lower(email));


--
-- Name: staff_members_company_id_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX staff_members_company_id_idx ON public.staff_members USING btree (company_id);


--
-- Name: users_company_id_idx; Type: INDEX; Schema: public; Owner: pulse
--

CREATE INDEX users_company_id_idx ON public.users USING btree (company_id);


--
-- Name: users_email_unique; Type: INDEX; Schema: public; Owner: pulse
--

CREATE UNIQUE INDEX users_email_unique ON public.users USING btree (lower(email));


--
-- Name: users_phone_unique; Type: INDEX; Schema: public; Owner: pulse
--

CREATE UNIQUE INDEX users_phone_unique ON public.users USING btree (phone) WHERE (phone IS NOT NULL);


--
-- Name: attendance_logs attendance_logs_booking_id_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.attendance_logs
    ADD CONSTRAINT attendance_logs_booking_id_company_id_fkey FOREIGN KEY (booking_id, company_id) REFERENCES public.bookings(id, company_id);


--
-- Name: attendance_logs attendance_logs_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.attendance_logs
    ADD CONSTRAINT attendance_logs_company_id_fkey FOREIGN KEY (company_id) REFERENCES public.companies(id) ON DELETE CASCADE;


--
-- Name: attendance_logs attendance_logs_recorded_by_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.attendance_logs
    ADD CONSTRAINT attendance_logs_recorded_by_company_id_fkey FOREIGN KEY (recorded_by, company_id) REFERENCES public.users(id, company_id);


--
-- Name: attendance_logs attendance_logs_staff_id_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.attendance_logs
    ADD CONSTRAINT attendance_logs_staff_id_company_id_fkey FOREIGN KEY (staff_id, company_id) REFERENCES public.staff_members(id, company_id) ON DELETE CASCADE;


--
-- Name: booking_history booking_history_actor_id_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.booking_history
    ADD CONSTRAINT booking_history_actor_id_company_id_fkey FOREIGN KEY (actor_id, company_id) REFERENCES public.users(id, company_id);


--
-- Name: booking_history booking_history_booking_id_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.booking_history
    ADD CONSTRAINT booking_history_booking_id_company_id_fkey FOREIGN KEY (booking_id, company_id) REFERENCES public.bookings(id, company_id) ON DELETE CASCADE;


--
-- Name: booking_history booking_history_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.booking_history
    ADD CONSTRAINT booking_history_company_id_fkey FOREIGN KEY (company_id) REFERENCES public.companies(id) ON DELETE CASCADE;


--
-- Name: bookings bookings_client_id_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.bookings
    ADD CONSTRAINT bookings_client_id_company_id_fkey FOREIGN KEY (client_id, company_id) REFERENCES public.clients(id, company_id);


--
-- Name: bookings bookings_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.bookings
    ADD CONSTRAINT bookings_company_id_fkey FOREIGN KEY (company_id) REFERENCES public.companies(id) ON DELETE CASCADE;


--
-- Name: bookings bookings_created_by_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.bookings
    ADD CONSTRAINT bookings_created_by_company_id_fkey FOREIGN KEY (created_by, company_id) REFERENCES public.users(id, company_id);


--
-- Name: clients clients_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.clients
    ADD CONSTRAINT clients_company_id_fkey FOREIGN KEY (company_id) REFERENCES public.companies(id) ON DELETE CASCADE;


--
-- Name: companies companies_reviewed_by_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.companies
    ADD CONSTRAINT companies_reviewed_by_fkey FOREIGN KEY (reviewed_by) REFERENCES public.platform_admins(id) ON DELETE SET NULL;


--
-- Name: face_embeddings face_embeddings_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.face_embeddings
    ADD CONSTRAINT face_embeddings_company_id_fkey FOREIGN KEY (company_id) REFERENCES public.companies(id) ON DELETE CASCADE;


--
-- Name: face_embeddings face_embeddings_staff_id_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.face_embeddings
    ADD CONSTRAINT face_embeddings_staff_id_company_id_fkey FOREIGN KEY (staff_id, company_id) REFERENCES public.staff_members(id, company_id) ON DELETE CASCADE;


--
-- Name: inventory_items inventory_items_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.inventory_items
    ADD CONSTRAINT inventory_items_company_id_fkey FOREIGN KEY (company_id) REFERENCES public.companies(id) ON DELETE CASCADE;


--
-- Name: item_movements item_movements_booking_id_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.item_movements
    ADD CONSTRAINT item_movements_booking_id_company_id_fkey FOREIGN KEY (booking_id, company_id) REFERENCES public.bookings(id, company_id);


--
-- Name: item_movements item_movements_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.item_movements
    ADD CONSTRAINT item_movements_company_id_fkey FOREIGN KEY (company_id) REFERENCES public.companies(id) ON DELETE CASCADE;


--
-- Name: item_movements item_movements_item_id_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.item_movements
    ADD CONSTRAINT item_movements_item_id_company_id_fkey FOREIGN KEY (item_id, company_id) REFERENCES public.inventory_items(id, company_id);


--
-- Name: item_movements item_movements_recorded_by_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.item_movements
    ADD CONSTRAINT item_movements_recorded_by_company_id_fkey FOREIGN KEY (recorded_by, company_id) REFERENCES public.users(id, company_id);


--
-- Name: staff_members staff_members_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.staff_members
    ADD CONSTRAINT staff_members_company_id_fkey FOREIGN KEY (company_id) REFERENCES public.companies(id) ON DELETE CASCADE;


--
-- Name: staff_members staff_members_user_id_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.staff_members
    ADD CONSTRAINT staff_members_user_id_company_id_fkey FOREIGN KEY (user_id, company_id) REFERENCES public.users(id, company_id);


--
-- Name: users users_company_id_fkey; Type: FK CONSTRAINT; Schema: public; Owner: pulse
--

ALTER TABLE ONLY public.users
    ADD CONSTRAINT users_company_id_fkey FOREIGN KEY (company_id) REFERENCES public.companies(id) ON DELETE CASCADE;


--
-- Name: attendance_logs; Type: ROW SECURITY; Schema: public; Owner: pulse
--

ALTER TABLE public.attendance_logs ENABLE ROW LEVEL SECURITY;

--
-- Name: booking_history; Type: ROW SECURITY; Schema: public; Owner: pulse
--

ALTER TABLE public.booking_history ENABLE ROW LEVEL SECURITY;

--
-- Name: bookings; Type: ROW SECURITY; Schema: public; Owner: pulse
--

ALTER TABLE public.bookings ENABLE ROW LEVEL SECURITY;

--
-- Name: clients; Type: ROW SECURITY; Schema: public; Owner: pulse
--

ALTER TABLE public.clients ENABLE ROW LEVEL SECURITY;

--
-- Name: companies; Type: ROW SECURITY; Schema: public; Owner: pulse
--

ALTER TABLE public.companies ENABLE ROW LEVEL SECURITY;

--
-- Name: attendance_logs company_isolation; Type: POLICY; Schema: public; Owner: pulse
--

CREATE POLICY company_isolation ON public.attendance_logs USING ((company_id = public.app_current_company())) WITH CHECK ((company_id = public.app_current_company()));


--
-- Name: booking_history company_isolation; Type: POLICY; Schema: public; Owner: pulse
--

CREATE POLICY company_isolation ON public.booking_history USING ((company_id = public.app_current_company())) WITH CHECK ((company_id = public.app_current_company()));


--
-- Name: bookings company_isolation; Type: POLICY; Schema: public; Owner: pulse
--

CREATE POLICY company_isolation ON public.bookings USING ((company_id = public.app_current_company())) WITH CHECK ((company_id = public.app_current_company()));


--
-- Name: clients company_isolation; Type: POLICY; Schema: public; Owner: pulse
--

CREATE POLICY company_isolation ON public.clients USING ((company_id = public.app_current_company())) WITH CHECK ((company_id = public.app_current_company()));


--
-- Name: companies company_isolation; Type: POLICY; Schema: public; Owner: pulse
--

CREATE POLICY company_isolation ON public.companies USING ((id = public.app_current_company())) WITH CHECK ((id = public.app_current_company()));


--
-- Name: face_embeddings company_isolation; Type: POLICY; Schema: public; Owner: pulse
--

CREATE POLICY company_isolation ON public.face_embeddings USING ((company_id = public.app_current_company())) WITH CHECK ((company_id = public.app_current_company()));


--
-- Name: inventory_items company_isolation; Type: POLICY; Schema: public; Owner: pulse
--

CREATE POLICY company_isolation ON public.inventory_items USING ((company_id = public.app_current_company())) WITH CHECK ((company_id = public.app_current_company()));


--
-- Name: item_movements company_isolation; Type: POLICY; Schema: public; Owner: pulse
--

CREATE POLICY company_isolation ON public.item_movements USING ((company_id = public.app_current_company())) WITH CHECK ((company_id = public.app_current_company()));


--
-- Name: staff_members company_isolation; Type: POLICY; Schema: public; Owner: pulse
--

CREATE POLICY company_isolation ON public.staff_members USING ((company_id = public.app_current_company())) WITH CHECK ((company_id = public.app_current_company()));


--
-- Name: users company_isolation; Type: POLICY; Schema: public; Owner: pulse
--

CREATE POLICY company_isolation ON public.users USING ((company_id = public.app_current_company())) WITH CHECK ((company_id = public.app_current_company()));


--
-- Name: face_embeddings; Type: ROW SECURITY; Schema: public; Owner: pulse
--

ALTER TABLE public.face_embeddings ENABLE ROW LEVEL SECURITY;

--
-- Name: inventory_items; Type: ROW SECURITY; Schema: public; Owner: pulse
--

ALTER TABLE public.inventory_items ENABLE ROW LEVEL SECURITY;

--
-- Name: item_movements; Type: ROW SECURITY; Schema: public; Owner: pulse
--

ALTER TABLE public.item_movements ENABLE ROW LEVEL SECURITY;

--
-- Name: staff_members; Type: ROW SECURITY; Schema: public; Owner: pulse
--

ALTER TABLE public.staff_members ENABLE ROW LEVEL SECURITY;

--
-- Name: users; Type: ROW SECURITY; Schema: public; Owner: pulse
--

ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;

--
-- Name: SCHEMA public; Type: ACL; Schema: -; Owner: pg_database_owner
--

GRANT USAGE ON SCHEMA public TO pulse_app;


--
-- Name: FUNCTION auth_find_user(p_email text, p_phone text); Type: ACL; Schema: public; Owner: pulse
--

REVOKE ALL ON FUNCTION public.auth_find_user(p_email text, p_phone text) FROM PUBLIC;
GRANT ALL ON FUNCTION public.auth_find_user(p_email text, p_phone text) TO pulse_app;


--
-- Name: FUNCTION auth_record_login(p_user uuid, p_success boolean, p_max integer, p_lock_minutes integer); Type: ACL; Schema: public; Owner: pulse
--

REVOKE ALL ON FUNCTION public.auth_record_login(p_user uuid, p_success boolean, p_max integer, p_lock_minutes integer) FROM PUBLIC;
GRANT ALL ON FUNCTION public.auth_record_login(p_user uuid, p_success boolean, p_max integer, p_lock_minutes integer) TO pulse_app;


--
-- Name: FUNCTION platform_find_admin(p_email text); Type: ACL; Schema: public; Owner: pulse
--

REVOKE ALL ON FUNCTION public.platform_find_admin(p_email text) FROM PUBLIC;
GRANT ALL ON FUNCTION public.platform_find_admin(p_email text) TO pulse_app;


--
-- Name: FUNCTION platform_get_admin(p_id uuid); Type: ACL; Schema: public; Owner: pulse
--

REVOKE ALL ON FUNCTION public.platform_get_admin(p_id uuid) FROM PUBLIC;
GRANT ALL ON FUNCTION public.platform_get_admin(p_id uuid) TO pulse_app;


--
-- Name: FUNCTION platform_list_companies(); Type: ACL; Schema: public; Owner: pulse
--

REVOKE ALL ON FUNCTION public.platform_list_companies() FROM PUBLIC;
GRANT ALL ON FUNCTION public.platform_list_companies() TO pulse_app;


--
-- Name: FUNCTION platform_set_company_status(p_company uuid, p_status text, p_note text, p_admin uuid); Type: ACL; Schema: public; Owner: pulse
--

REVOKE ALL ON FUNCTION public.platform_set_company_status(p_company uuid, p_status text, p_note text, p_admin uuid) FROM PUBLIC;
GRANT ALL ON FUNCTION public.platform_set_company_status(p_company uuid, p_status text, p_note text, p_admin uuid) TO pulse_app;


--
-- Name: TABLE attendance_logs; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.attendance_logs TO pulse_app;


--
-- Name: TABLE booking_history; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.booking_history TO pulse_app;


--
-- Name: TABLE bookings; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.bookings TO pulse_app;


--
-- Name: TABLE clients; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.clients TO pulse_app;


--
-- Name: TABLE companies; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,DELETE ON TABLE public.companies TO pulse_app;


--
-- Name: COLUMN companies.id; Type: ACL; Schema: public; Owner: pulse
--

GRANT INSERT(id) ON TABLE public.companies TO pulse_app;


--
-- Name: COLUMN companies.name; Type: ACL; Schema: public; Owner: pulse
--

GRANT INSERT(name),UPDATE(name) ON TABLE public.companies TO pulse_app;


--
-- Name: COLUMN companies.phone; Type: ACL; Schema: public; Owner: pulse
--

GRANT INSERT(phone),UPDATE(phone) ON TABLE public.companies TO pulse_app;


--
-- Name: COLUMN companies.address; Type: ACL; Schema: public; Owner: pulse
--

GRANT INSERT(address),UPDATE(address) ON TABLE public.companies TO pulse_app;


--
-- Name: COLUMN companies.city; Type: ACL; Schema: public; Owner: pulse
--

GRANT INSERT(city),UPDATE(city) ON TABLE public.companies TO pulse_app;


--
-- Name: COLUMN companies.logo_url; Type: ACL; Schema: public; Owner: pulse
--

GRANT INSERT(logo_url),UPDATE(logo_url) ON TABLE public.companies TO pulse_app;


--
-- Name: COLUMN companies.timezone; Type: ACL; Schema: public; Owner: pulse
--

GRANT INSERT(timezone),UPDATE(timezone) ON TABLE public.companies TO pulse_app;


--
-- Name: TABLE face_embeddings; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.face_embeddings TO pulse_app;


--
-- Name: TABLE inventory_items; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.inventory_items TO pulse_app;


--
-- Name: TABLE item_movements; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.item_movements TO pulse_app;


--
-- Name: TABLE platform_admins; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.platform_admins TO pulse_app;


--
-- Name: TABLE staff_members; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.staff_members TO pulse_app;


--
-- Name: TABLE users; Type: ACL; Schema: public; Owner: pulse
--

GRANT SELECT,INSERT,DELETE,UPDATE ON TABLE public.users TO pulse_app;


--
-- Name: DEFAULT PRIVILEGES FOR SEQUENCES; Type: DEFAULT ACL; Schema: public; Owner: pulse
--

ALTER DEFAULT PRIVILEGES FOR ROLE pulse IN SCHEMA public GRANT SELECT,USAGE ON SEQUENCES TO pulse_app;


--
-- Name: DEFAULT PRIVILEGES FOR TABLES; Type: DEFAULT ACL; Schema: public; Owner: pulse
--

ALTER DEFAULT PRIVILEGES FOR ROLE pulse IN SCHEMA public GRANT SELECT,INSERT,DELETE,UPDATE ON TABLES TO pulse_app;


--
-- PostgreSQL database dump complete
--

\unrestrict U4sME4TDTrHXSmfG8V6ylb6AJUOYttxl8cvfnG7SEChH1wDdosaBY8kkvK3VaFT

