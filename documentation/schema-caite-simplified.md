-- CAITE MVP Database Schema (Simplified)
-- Focus on core functionality for initial launch

-- ========================================
-- ENUMS (Essential Only)
-- ========================================

CREATE TYPE session_status_enum AS ENUM ('draft', 'published', 'archived');
CREATE TYPE threat_category_enum AS ENUM (
  'prompt_injection', 
  'data_exfiltration', 
  'jailbreak_attempt',
  'bias_evaluation',
  'capability_testing',
  'creative_use',
  'other'
);

-- ========================================
-- CORE TABLES (MVP)
-- ========================================

-- User profiles (minimal extension of Supabase auth)
CREATE TABLE public.users (
  id uuid PRIMARY KEY,
  username text UNIQUE NOT NULL,
  display_name text,
  bio text,
  avatar_url text,
  is_verified boolean DEFAULT false,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),
  CONSTRAINT users_id_fkey FOREIGN KEY (id) REFERENCES auth.users(id) ON DELETE CASCADE
);

-- Chat sessions (core functionality only)
CREATE TABLE public.chat_sessions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
  title text NOT NULL,
  description text,
  slug text NOT NULL,
  category threat_category_enum NOT NULL DEFAULT 'other',
  tags text[] DEFAULT ARRAY[]::text[],
  seo_keywords text[] DEFAULT ARRAY[]::text[],
  llm_model text NOT NULL,
  provider text, -- openai, anthropic, etc.
  status session_status_enum DEFAULT 'draft',
  view_count integer DEFAULT 0,
  published_at timestamptz,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now(),
  UNIQUE(user_id, slug)
);

-- Chat messages (essential fields only)
CREATE TABLE public.chat_messages (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id uuid NOT NULL REFERENCES public.chat_sessions(id) ON DELETE CASCADE,
  role text NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
  content jsonb NOT NULL, -- Flexible content: text, images, etc.
  message_index integer NOT NULL,
  metadata jsonb DEFAULT '{}', -- Tool calls, reasoning, etc.
  created_at timestamptz DEFAULT now(),
  UNIQUE(session_id, message_index)
);

-- Session views (basic analytics)
CREATE TABLE public.session_views (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id uuid NOT NULL REFERENCES public.chat_sessions(id) ON DELETE CASCADE,
  viewer_ip inet,
  viewed_at timestamptz DEFAULT now()
);

-- ========================================
-- INDEXES (Essential Only)
-- ========================================

CREATE INDEX idx_chat_sessions_published ON public.chat_sessions(status, published_at DESC) WHERE status = 'published';
CREATE INDEX idx_chat_sessions_category ON public.chat_sessions(category);
CREATE INDEX idx_chat_sessions_user ON public.chat_sessions(user_id);
CREATE INDEX idx_chat_sessions_slug ON public.chat_sessions(user_id, slug);

-- Full-text search
CREATE INDEX idx_chat_sessions_search ON public.chat_sessions USING gin(to_tsvector('english', title || ' ' || COALESCE(description, '')));
CREATE INDEX idx_chat_messages_search ON public.chat_messages USING gin(to_tsvector('english', content::text));

-- Analytics
CREATE INDEX idx_session_views_session_id ON public.session_views(session_id);

-- ========================================
-- ROW LEVEL SECURITY (Minimal)
-- ========================================

ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.chat_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.chat_messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.session_views ENABLE ROW LEVEL SECURITY;

-- Users: Public profiles
CREATE POLICY "Public user profiles" ON public.users FOR SELECT USING (true);
CREATE POLICY "Users update own profile" ON public.users FOR ALL USING (auth.uid() = id);

-- Sessions: Published are public, owners see all
CREATE POLICY "Published sessions public" ON public.chat_sessions FOR SELECT 
  USING (status = 'published' OR auth.uid() = user_id);

CREATE POLICY "Users manage own sessions" ON public.chat_sessions FOR ALL 
  USING (auth.uid() = user_id) WITH CHECK (auth.uid() = user_id);

-- Messages: Follow session visibility
CREATE POLICY "Messages follow session policy" ON public.chat_messages FOR SELECT 
  USING (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND (status = 'published' OR user_id = auth.uid())
    )
  );

CREATE POLICY "Users manage own messages" ON public.chat_messages FOR ALL 
  USING (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND user_id = auth.uid()
    )
  ) WITH CHECK (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND user_id = auth.uid()
    )
  );

-- Views: Anyone can record, owners can read
CREATE POLICY "Anyone record views" ON public.session_views FOR INSERT WITH CHECK (true);
CREATE POLICY "Session owners view analytics" ON public.session_views FOR SELECT 
  USING (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND user_id = auth.uid()
    )
  );

-- ========================================
-- TRIGGERS (Essential Only)
-- ========================================

-- Auto-increment view count
CREATE OR REPLACE FUNCTION increment_view_count()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE public.chat_sessions 
  SET view_count = view_count + 1 
  WHERE id = NEW.session_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_increment_view_count
  AFTER INSERT ON public.session_views
  FOR EACH ROW EXECUTE FUNCTION increment_view_count();

-- Auto-update timestamps
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_users_updated_at
  BEFORE UPDATE ON public.users FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER trigger_sessions_updated_at
  BEFORE UPDATE ON public.chat_sessions FOR EACH ROW
  EXECUTE FUNCTION update_updated_at();

-- ========================================
-- SEED DATA
-- ========================================

-- Essential categories only
INSERT INTO public.users (id, username, display_name, is_verified)
SELECT 
  '00000000-0000-0000-0000-000000000000'::uuid,
  'caite_admin', 
  'CAITE Admin', 
  true
WHERE NOT EXISTS (SELECT 1 FROM public.users WHERE username = 'caite_admin');

/*
SIMPLIFICATION SUMMARY:
======================

REMOVED FEATURES (Add later when needed):
- Social features (follows, comments, likes, bookmarks)
- Expert review system
- Moderation/reporting system
- API tokens and social sharing
- Complex analytics beyond view counts
- Notification preferences
- Security logging
- Content flagging workflow

SIMPLIFIED TABLES:
- users: 8 fields (was 15+)
- chat_sessions: 15 fields (was 25+)
- chat_messages: 7 fields (was 10+)
- session_views: 4 fields (was 8+)

TOTAL: 4 tables (was 16 tables)

MVP CAPABILITIES RETAINED:
✅ User profiles with usernames
✅ Chat session storage with categorization  
✅ SEO-friendly slug URLs (/user/username/slug)
✅ Threat categorization system
✅ Full-text search capabilities
✅ Basic analytics (view counts)
✅ Draft/published workflow
✅ Permanent PostgreSQL storage
✅ Row Level Security for privacy

PHASE 2 ADDITIONS (when product validates):
- Community features (comments, follows, likes)
- Expert review system  
- Advanced moderation tools
- API access and integrations
- Enhanced analytics
- Social sharing tracking
*/