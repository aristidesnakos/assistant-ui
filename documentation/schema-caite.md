-- CAITE (Chat AI Threat Evidence) Database Schema
-- WARNING: This schema is for context only and is not meant to be run directly.
-- Execute in proper order with dependency management for production use.

-- ========================================
-- ENUMS AND TYPES
-- ========================================

-- Chat session status lifecycle
CREATE TYPE session_status_enum AS ENUM ('draft', 'published', 'archived', 'flagged', 'under_review');

-- Threat level classification
CREATE TYPE threat_level_enum AS ENUM ('low', 'medium', 'high', 'critical');

-- Message roles in chat conversations
CREATE TYPE message_role_enum AS ENUM ('user', 'assistant', 'system', 'tool');

-- Security categories for AI threat intelligence
CREATE TYPE security_category_enum AS ENUM (
  'prompt_injection', 
  'data_exfiltration', 
  'jailbreak_attempt',
  'bias_evaluation',
  'capability_testing',
  'creative_use',
  'hallucination',
  'privacy_violation',
  'misinformation',
  'other'
);

-- ========================================
-- CORE TABLES
-- ========================================

-- User profiles (extends Supabase auth.users)
CREATE TABLE public.users (
  id uuid NOT NULL,
  username text UNIQUE NOT NULL,
  display_name text,
  bio text,
  avatar_url text,
  website_url text,
  github_url text,
  twitter_url text,
  linkedin_url text,
  organization text,
  location text,
  is_verified boolean DEFAULT false,
  is_researcher boolean DEFAULT false,
  privacy_settings jsonb DEFAULT '{
    "show_email": false,
    "show_real_name": false,
    "allow_contact": true,
    "public_sessions": true
  }'::jsonb,
  notification_preferences jsonb DEFAULT '{
    "email_mentions": true,
    "email_follows": true,
    "email_featured": true
  }'::jsonb,
  created_at timestamp with time zone NOT NULL DEFAULT timezone('utc'::text, now()),
  updated_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  CONSTRAINT users_pkey PRIMARY KEY (id),
  CONSTRAINT users_id_fkey FOREIGN KEY (id) REFERENCES auth.users(id) ON DELETE CASCADE
);

-- Chat sessions - core entity for threat intelligence documentation
CREATE TABLE public.chat_sessions (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL,
  title text NOT NULL,
  description text,
  slug text NOT NULL,
  category security_category_enum NOT NULL DEFAULT 'other',
  subcategories text[] DEFAULT ARRAY[]::text[],
  seo_keywords text[] DEFAULT ARRAY[]::text[],
  threat_level threat_level_enum DEFAULT 'low',
  llm_model text NOT NULL,
  llm_version text,
  provider text, -- openai, anthropic, google, etc.
  session_metadata jsonb DEFAULT '{}'::jsonb,
  status session_status_enum DEFAULT 'draft',
  featured boolean DEFAULT false,
  view_count integer DEFAULT 0,
  like_count integer DEFAULT 0,
  bookmark_count integer DEFAULT 0,
  share_count integer DEFAULT 0,
  published_at timestamp with time zone,
  flagged_at timestamp with time zone,
  flag_reason text,
  moderator_notes text,
  original_conversation_url text, -- Link to original platform if applicable
  research_notes text,
  mitigation_strategies text,
  impact_assessment text,
  reproduced boolean DEFAULT false,
  reproduction_notes text,
  cve_references text[], -- Related CVE identifiers if applicable
  external_references text[], -- Links to papers, articles, etc.
  tags text[] DEFAULT ARRAY[]::text[],
  created_at timestamp with time zone NOT NULL DEFAULT timezone('utc'::text, now()),
  updated_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  CONSTRAINT chat_sessions_pkey PRIMARY KEY (id),
  CONSTRAINT chat_sessions_user_id_fkey FOREIGN KEY (user_id) REFERENCES public.users(id) ON DELETE CASCADE,
  CONSTRAINT unique_user_slug UNIQUE (user_id, slug)
);

-- Individual messages within chat sessions
CREATE TABLE public.chat_messages (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  session_id uuid NOT NULL,
  role message_role_enum NOT NULL,
  content jsonb NOT NULL, -- Flexible content structure for text, images, files
  message_index integer NOT NULL,
  timestamp_in_conversation timestamp with time zone,
  metadata jsonb DEFAULT '{}'::jsonb, -- Tool calls, reasoning steps, attachments, etc.
  token_count integer,
  processing_time_ms integer,
  flagged boolean DEFAULT false,
  flag_reason text,
  annotations jsonb DEFAULT '{}'::jsonb, -- User annotations and analysis
  created_at timestamp with time zone NOT NULL DEFAULT timezone('utc'::text, now()),
  CONSTRAINT chat_messages_pkey PRIMARY KEY (id),
  CONSTRAINT chat_messages_session_id_fkey FOREIGN KEY (session_id) REFERENCES public.chat_sessions(id) ON DELETE CASCADE,
  CONSTRAINT unique_session_message_index UNIQUE (session_id, message_index)
);

-- ========================================
-- CATEGORIZATION AND TAXONOMY
-- ========================================

-- Predefined categories with metadata
CREATE TABLE public.categories (
  slug text PRIMARY KEY,
  name text NOT NULL,
  description text NOT NULL,
  color text NOT NULL DEFAULT '#gray',
  icon text,
  subcategories text[] NOT NULL DEFAULT ARRAY[]::text[],
  parent_category text REFERENCES public.categories(slug),
  is_active boolean DEFAULT true,
  sort_order integer DEFAULT 0,
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  updated_at timestamp with time zone DEFAULT timezone('utc'::text, now())
);

-- User-defined tags for flexible categorization
CREATE TABLE public.tags (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  name text UNIQUE NOT NULL,
  description text,
  color text DEFAULT '#blue',
  usage_count integer DEFAULT 0,
  created_by uuid REFERENCES public.users(id),
  is_verified boolean DEFAULT false,
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  CONSTRAINT tags_pkey PRIMARY KEY (id)
);

-- Many-to-many relationship between sessions and tags
CREATE TABLE public.session_tags (
  session_id uuid REFERENCES public.chat_sessions(id) ON DELETE CASCADE,
  tag_id uuid REFERENCES public.tags(id) ON DELETE CASCADE,
  added_by uuid REFERENCES public.users(id),
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  PRIMARY KEY (session_id, tag_id)
);

-- ========================================
-- ANALYTICS AND ENGAGEMENT
-- ========================================

-- Session view tracking for analytics
CREATE TABLE public.session_views (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  session_id uuid NOT NULL,
  viewer_id uuid REFERENCES public.users(id), -- null for anonymous
  viewer_ip inet,
  user_agent text,
  referrer text,
  country_code text,
  device_type text, -- mobile, desktop, tablet
  session_duration integer, -- seconds spent viewing
  viewed_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  CONSTRAINT session_views_pkey PRIMARY KEY (id),
  CONSTRAINT session_views_session_id_fkey FOREIGN KEY (session_id) REFERENCES public.chat_sessions(id) ON DELETE CASCADE
);

-- User likes/reactions to sessions
CREATE TABLE public.session_likes (
  user_id uuid REFERENCES public.users(id) ON DELETE CASCADE,
  session_id uuid REFERENCES public.chat_sessions(id) ON DELETE CASCADE,
  reaction_type text DEFAULT 'like' CHECK (reaction_type IN ('like', 'helpful', 'concerning', 'bookmark')),
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  PRIMARY KEY (user_id, session_id, reaction_type)
);

-- User bookmarks for saving interesting sessions
CREATE TABLE public.session_bookmarks (
  user_id uuid REFERENCES public.users(id) ON DELETE CASCADE,
  session_id uuid REFERENCES public.chat_sessions(id) ON DELETE CASCADE,
  folder_name text DEFAULT 'default',
  notes text,
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  PRIMARY KEY (user_id, session_id)
);

-- ========================================
-- COMMUNITY AND COLLABORATION
-- ========================================

-- User follows for building research communities
CREATE TABLE public.user_follows (
  follower_id uuid REFERENCES public.users(id) ON DELETE CASCADE,
  following_id uuid REFERENCES public.users(id) ON DELETE CASCADE,
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  PRIMARY KEY (follower_id, following_id),
  CONSTRAINT no_self_follow CHECK (follower_id != following_id)
);

-- Comments on sessions for community discussion
CREATE TABLE public.session_comments (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  session_id uuid NOT NULL,
  user_id uuid NOT NULL,
  parent_comment_id uuid REFERENCES public.session_comments(id) ON DELETE CASCADE,
  content text NOT NULL,
  is_expert_opinion boolean DEFAULT false,
  helpful_count integer DEFAULT 0,
  flagged boolean DEFAULT false,
  flag_reason text,
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  updated_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  CONSTRAINT session_comments_pkey PRIMARY KEY (id),
  CONSTRAINT session_comments_session_id_fkey FOREIGN KEY (session_id) REFERENCES public.chat_sessions(id) ON DELETE CASCADE,
  CONSTRAINT session_comments_user_id_fkey FOREIGN KEY (user_id) REFERENCES public.users(id) ON DELETE CASCADE
);

-- Expert reviews and validations
CREATE TABLE public.expert_reviews (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  session_id uuid NOT NULL,
  expert_id uuid NOT NULL,
  review_type text CHECK (review_type IN ('validation', 'analysis', 'correction', 'enhancement')),
  severity_assessment threat_level_enum,
  technical_accuracy integer CHECK (technical_accuracy >= 1 AND technical_accuracy <= 5),
  reproducibility_confirmed boolean,
  review_notes text NOT NULL,
  recommendations text,
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  updated_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  CONSTRAINT expert_reviews_pkey PRIMARY KEY (id),
  CONSTRAINT expert_reviews_session_id_fkey FOREIGN KEY (session_id) REFERENCES public.chat_sessions(id) ON DELETE CASCADE,
  CONSTRAINT expert_reviews_expert_id_fkey FOREIGN KEY (expert_id) REFERENCES public.users(id) ON DELETE CASCADE,
  CONSTRAINT unique_expert_session_review UNIQUE (session_id, expert_id)
);

-- ========================================
-- SECURITY AND MODERATION
-- ========================================

-- Security incident logs for tracking malicious activity
CREATE TABLE public.security_logs (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES public.users(id),
  session_id uuid REFERENCES public.chat_sessions(id),
  event_type text NOT NULL, -- login_attempt, content_flagged, spam_detected, etc.
  severity text CHECK (severity IN ('low', 'medium', 'high', 'critical')),
  ip_address inet,
  user_agent text,
  event_data jsonb DEFAULT '{}'::jsonb,
  action_taken text,
  moderator_id uuid REFERENCES public.users(id),
  resolved_at timestamp with time zone,
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  CONSTRAINT security_logs_pkey PRIMARY KEY (id)
);

-- Content moderation reports
CREATE TABLE public.moderation_reports (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  reporter_id uuid REFERENCES public.users(id),
  session_id uuid REFERENCES public.chat_sessions(id),
  comment_id uuid REFERENCES public.session_comments(id),
  report_type text NOT NULL CHECK (report_type IN (
    'spam', 'misinformation', 'harmful_content', 'copyright', 
    'privacy_violation', 'inappropriate', 'other'
  )),
  description text NOT NULL,
  status text DEFAULT 'pending' CHECK (status IN ('pending', 'reviewing', 'resolved', 'dismissed')),
  moderator_id uuid REFERENCES public.users(id),
  moderator_notes text,
  action_taken text,
  resolved_at timestamp with time zone,
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  CONSTRAINT moderation_reports_pkey PRIMARY KEY (id)
);

-- ========================================
-- INTEGRATIONS AND EXPORTS
-- ========================================

-- API access tokens for external integrations
CREATE TABLE public.api_tokens (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL,
  token_name text NOT NULL,
  token_hash text NOT NULL, -- Store hashed version
  permissions text[] DEFAULT ARRAY['read']::text[], -- read, write, admin
  rate_limit integer DEFAULT 1000, -- requests per hour
  expires_at timestamp with time zone,
  last_used_at timestamp with time zone,
  created_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  CONSTRAINT api_tokens_pkey PRIMARY KEY (id),
  CONSTRAINT api_tokens_user_id_fkey FOREIGN KEY (user_id) REFERENCES public.users(id) ON DELETE CASCADE
);

-- External platform publications (Twitter, LinkedIn, etc.)
CREATE TABLE public.social_shares (
  id uuid NOT NULL DEFAULT gen_random_uuid(),
  session_id uuid NOT NULL,
  user_id uuid NOT NULL,
  platform text NOT NULL CHECK (platform IN ('twitter', 'linkedin', 'reddit', 'hackernews', 'github')),
  post_url text,
  post_id text,
  engagement_metrics jsonb DEFAULT '{}'::jsonb, -- likes, retweets, comments
  shared_at timestamp with time zone DEFAULT timezone('utc'::text, now()),
  CONSTRAINT social_shares_pkey PRIMARY KEY (id),
  CONSTRAINT social_shares_session_id_fkey FOREIGN KEY (session_id) REFERENCES public.chat_sessions(id) ON DELETE CASCADE,
  CONSTRAINT social_shares_user_id_fkey FOREIGN KEY (user_id) REFERENCES public.users(id) ON DELETE CASCADE
);

-- ========================================
-- INDEXES FOR PERFORMANCE
-- ========================================

-- Core query indexes
CREATE INDEX idx_chat_sessions_status_published ON public.chat_sessions(status, published_at DESC) WHERE status = 'published';
CREATE INDEX idx_chat_sessions_category ON public.chat_sessions(category);
CREATE INDEX idx_chat_sessions_user_id ON public.chat_sessions(user_id);
CREATE INDEX idx_chat_sessions_featured ON public.chat_sessions(featured, published_at DESC) WHERE featured = true;
CREATE INDEX idx_chat_sessions_threat_level ON public.chat_sessions(threat_level);

-- Search indexes
CREATE INDEX idx_chat_sessions_title_search ON public.chat_sessions USING gin(to_tsvector('english', title));
CREATE INDEX idx_chat_sessions_description_search ON public.chat_sessions USING gin(to_tsvector('english', description));
CREATE INDEX idx_chat_sessions_tags_search ON public.chat_sessions USING gin(tags);
CREATE INDEX idx_chat_sessions_keywords_search ON public.chat_sessions USING gin(seo_keywords);

-- Message content search
CREATE INDEX idx_chat_messages_content_search ON public.chat_messages USING gin(to_tsvector('english', content::text));

-- Analytics indexes
CREATE INDEX idx_session_views_session_id ON public.session_views(session_id);
CREATE INDEX idx_session_views_viewed_at ON public.session_views(viewed_at);
CREATE INDEX idx_session_likes_session_id ON public.session_likes(session_id);

-- User activity indexes
CREATE INDEX idx_users_username ON public.users(username);
CREATE INDEX idx_users_is_verified ON public.users(is_verified) WHERE is_verified = true;
CREATE INDEX idx_users_is_researcher ON public.users(is_researcher) WHERE is_researcher = true;

-- ========================================
-- ROW LEVEL SECURITY (RLS) POLICIES
-- ========================================

-- Enable RLS on all tables
ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.chat_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.chat_messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.session_views ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.session_likes ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.session_bookmarks ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.user_follows ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.session_comments ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.expert_reviews ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.security_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.moderation_reports ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.api_tokens ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.social_shares ENABLE ROW LEVEL SECURITY;

-- ========================================
-- USER POLICIES
-- ========================================

-- Public profiles are viewable by everyone
CREATE POLICY "Public user profiles are viewable by everyone" 
  ON public.users FOR SELECT 
  USING (true);

-- Users can update their own profile
CREATE POLICY "Users can update own profile" 
  ON public.users FOR UPDATE 
  USING (auth.uid() = id);

-- Users can insert their own profile (during onboarding)
CREATE POLICY "Users can insert own profile" 
  ON public.users FOR INSERT 
  WITH CHECK (auth.uid() = id);

-- ========================================
-- CHAT SESSION POLICIES
-- ========================================

-- Published sessions are viewable by everyone
CREATE POLICY "Published sessions are viewable by everyone" 
  ON public.chat_sessions FOR SELECT 
  USING (status = 'published');

-- Users can view their own sessions (all statuses)
CREATE POLICY "Users can view their own sessions" 
  ON public.chat_sessions FOR SELECT 
  USING (auth.uid() = user_id);

-- Moderators can view flagged sessions
CREATE POLICY "Moderators can view flagged sessions" 
  ON public.chat_sessions FOR SELECT 
  USING (
    status = 'flagged' AND 
    EXISTS (
      SELECT 1 FROM public.users 
      WHERE id = auth.uid() 
        AND (is_verified = true OR id IN (
          SELECT user_id FROM public.expert_reviews 
          GROUP BY user_id 
          HAVING COUNT(*) >= 5
        ))
    )
  );

-- Users can insert their own sessions
CREATE POLICY "Users can insert their own sessions" 
  ON public.chat_sessions FOR INSERT 
  WITH CHECK (auth.uid() = user_id);

-- Users can update their own sessions
CREATE POLICY "Users can update their own sessions" 
  ON public.chat_sessions FOR UPDATE 
  USING (auth.uid() = user_id);

-- Users can delete their own draft sessions
CREATE POLICY "Users can delete own draft sessions" 
  ON public.chat_sessions FOR DELETE 
  USING (auth.uid() = user_id AND status = 'draft');

-- ========================================
-- CHAT MESSAGE POLICIES
-- ========================================

-- Messages are visible based on session visibility rules
CREATE POLICY "Messages visible based on session policy" 
  ON public.chat_messages FOR SELECT 
  USING (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id 
        AND (
          status = 'published' 
          OR user_id = auth.uid()
          OR (status = 'flagged' AND EXISTS (
            SELECT 1 FROM public.users 
            WHERE id = auth.uid() AND is_verified = true
          ))
        )
    )
  );

-- Users can insert messages to their own sessions
CREATE POLICY "Users can insert messages to own sessions" 
  ON public.chat_messages FOR INSERT 
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND user_id = auth.uid()
    )
  );

-- Users can update messages in their own sessions
CREATE POLICY "Users can update messages in own sessions" 
  ON public.chat_messages FOR UPDATE 
  USING (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND user_id = auth.uid()
    )
  );

-- ========================================
-- ANALYTICS POLICIES
-- ========================================

-- Anyone can insert view records (for analytics)
CREATE POLICY "Anyone can record session views" 
  ON public.session_views FOR INSERT 
  WITH CHECK (true);

-- Users can view analytics for their own sessions
CREATE POLICY "Users can view analytics for own sessions" 
  ON public.session_views FOR SELECT 
  USING (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND user_id = auth.uid()
    )
  );

-- Global analytics visible to verified researchers
CREATE POLICY "Verified researchers can view aggregated analytics" 
  ON public.session_views FOR SELECT 
  USING (
    EXISTS (
      SELECT 1 FROM public.users 
      WHERE id = auth.uid() AND is_researcher = true
    )
  );

-- ========================================
-- ENGAGEMENT POLICIES
-- ========================================

-- Users can like/unlike any published session
CREATE POLICY "Users can react to published sessions" 
  ON public.session_likes FOR ALL 
  USING (
    auth.uid() = user_id AND
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND status = 'published'
    )
  )
  WITH CHECK (
    auth.uid() = user_id AND
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND status = 'published'
    )
  );

-- Users can bookmark any accessible session
CREATE POLICY "Users can bookmark accessible sessions" 
  ON public.session_bookmarks FOR ALL 
  USING (
    auth.uid() = user_id AND
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id 
        AND (status = 'published' OR user_id = auth.uid())
    )
  )
  WITH CHECK (
    auth.uid() = user_id AND
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id 
        AND (status = 'published' OR user_id = auth.uid())
    )
  );

-- ========================================
-- COMMUNITY POLICIES
-- ========================================

-- Users can follow/unfollow other users
CREATE POLICY "Users can manage their follows" 
  ON public.user_follows FOR ALL 
  USING (auth.uid() = follower_id)
  WITH CHECK (auth.uid() = follower_id);

-- Users can view who follows them and who they follow
CREATE POLICY "Users can view follow relationships" 
  ON public.user_follows FOR SELECT 
  USING (auth.uid() = follower_id OR auth.uid() = following_id);

-- Users can comment on published sessions
CREATE POLICY "Users can comment on published sessions" 
  ON public.session_comments FOR INSERT 
  WITH CHECK (
    auth.uid() = user_id AND
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND status = 'published'
    )
  );

-- Users can view comments on accessible sessions
CREATE POLICY "Comments visible on accessible sessions" 
  ON public.session_comments FOR SELECT 
  USING (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id 
        AND (
          status = 'published' 
          OR user_id = auth.uid()
        )
    )
  );

-- Users can update/delete their own comments
CREATE POLICY "Users can manage their own comments" 
  ON public.session_comments FOR UPDATE 
  USING (auth.uid() = user_id);

CREATE POLICY "Users can delete their own comments" 
  ON public.session_comments FOR DELETE 
  USING (auth.uid() = user_id);

-- ========================================
-- EXPERT REVIEW POLICIES
-- ========================================

-- Verified users can submit expert reviews
CREATE POLICY "Verified users can submit expert reviews" 
  ON public.expert_reviews FOR INSERT 
  WITH CHECK (
    auth.uid() = expert_id AND
    EXISTS (
      SELECT 1 FROM public.users 
      WHERE id = auth.uid() AND is_verified = true
    ) AND
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND status = 'published'
    )
  );

-- Expert reviews are publicly visible
CREATE POLICY "Expert reviews are publicly visible" 
  ON public.expert_reviews FOR SELECT 
  USING (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND status = 'published'
    )
  );

-- Experts can update their own reviews
CREATE POLICY "Experts can update own reviews" 
  ON public.expert_reviews FOR UPDATE 
  USING (auth.uid() = expert_id);

-- ========================================
-- MODERATION POLICIES
-- ========================================

-- Users can report content
CREATE POLICY "Users can submit moderation reports" 
  ON public.moderation_reports FOR INSERT 
  WITH CHECK (auth.uid() = reporter_id);

-- Users can view their own reports
CREATE POLICY "Users can view own reports" 
  ON public.moderation_reports FOR SELECT 
  USING (auth.uid() = reporter_id);

-- Moderators can view and manage all reports
CREATE POLICY "Moderators can manage reports" 
  ON public.moderation_reports FOR ALL 
  USING (
    EXISTS (
      SELECT 1 FROM public.users 
      WHERE id = auth.uid() AND is_verified = true
    )
  )
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM public.users 
      WHERE id = auth.uid() AND is_verified = true
    )
  );

-- Security logs only visible to moderators
CREATE POLICY "Security logs visible to moderators" 
  ON public.security_logs FOR SELECT 
  USING (
    EXISTS (
      SELECT 1 FROM public.users 
      WHERE id = auth.uid() AND is_verified = true
    )
  );

-- ========================================
-- API TOKEN POLICIES
-- ========================================

-- Users can manage their own API tokens
CREATE POLICY "Users can manage own API tokens" 
  ON public.api_tokens FOR ALL 
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- ========================================
-- SOCIAL SHARING POLICIES
-- ========================================

-- Users can share their own sessions or published sessions
CREATE POLICY "Users can share accessible sessions" 
  ON public.social_shares FOR INSERT 
  WITH CHECK (
    auth.uid() = user_id AND
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id 
        AND (status = 'published' OR user_id = auth.uid())
    )
  );

-- Social shares are publicly visible for published sessions
CREATE POLICY "Social shares visible for published sessions" 
  ON public.social_shares FOR SELECT 
  USING (
    EXISTS (
      SELECT 1 FROM public.chat_sessions 
      WHERE id = session_id AND status = 'published'
    )
  );

-- ========================================
-- FUNCTIONS AND TRIGGERS
-- ========================================

-- Function to update view counts
CREATE OR REPLACE FUNCTION increment_view_count()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE public.chat_sessions 
  SET view_count = view_count + 1 
  WHERE id = NEW.session_id;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger to automatically update view counts
CREATE TRIGGER trigger_increment_view_count
  AFTER INSERT ON public.session_views
  FOR EACH ROW
  EXECUTE FUNCTION increment_view_count();

-- Function to update like counts
CREATE OR REPLACE FUNCTION update_like_count()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE public.chat_sessions 
    SET like_count = like_count + 1 
    WHERE id = NEW.session_id;
    RETURN NEW;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE public.chat_sessions 
    SET like_count = like_count - 1 
    WHERE id = OLD.session_id;
    RETURN OLD;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Triggers for like count updates
CREATE TRIGGER trigger_update_like_count_insert
  AFTER INSERT ON public.session_likes
  FOR EACH ROW
  EXECUTE FUNCTION update_like_count();

CREATE TRIGGER trigger_update_like_count_delete
  AFTER DELETE ON public.session_likes
  FOR EACH ROW
  EXECUTE FUNCTION update_like_count();

-- Function to automatically update updated_at timestamps
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = timezone('utc'::text, now());
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Triggers to update timestamps
CREATE TRIGGER trigger_users_updated_at
  BEFORE UPDATE ON public.users
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trigger_chat_sessions_updated_at
  BEFORE UPDATE ON public.chat_sessions
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trigger_session_comments_updated_at
  BEFORE UPDATE ON public.session_comments
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

-- ========================================
-- INITIAL DATA SEEDING
-- ========================================

-- Insert default categories
INSERT INTO public.categories (slug, name, description, color, subcategories) VALUES
('prompt_injection', 'Prompt Injection', 'Attempts to manipulate LLM behavior through crafted inputs', '#ef4444', ARRAY['direct', 'indirect', 'context_overflow', 'delimiter_injection']),
('data_exfiltration', 'Data Exfiltration', 'Attempts to extract training data or sensitive information', '#f97316', ARRAY['training_data', 'pii_leak', 'code_extraction', 'system_prompt_leak']),
('jailbreak_attempt', 'Jailbreak Attempt', 'Attempts to bypass safety restrictions and guidelines', '#dc2626', ARRAY['role_play', 'hypothetical', 'character_bypass', 'emotional_manipulation']),
('bias_evaluation', 'Bias Evaluation', 'Testing for biased or harmful outputs', '#8b5cf6', ARRAY['gender_bias', 'racial_bias', 'cultural_bias', 'political_bias']),
('capability_testing', 'Capability Testing', 'Exploring LLM capabilities and limitations', '#3b82f6', ARRAY['reasoning', 'knowledge_boundaries', 'multimodal', 'coding_abilities']),
('creative_use', 'Creative Use', 'Novel or innovative use cases for LLMs', '#10b981', ARRAY['art_generation', 'creative_writing', 'problem_solving', 'educational_tools']),
('hallucination', 'Hallucination', 'Documenting factual errors and false information generation', '#f59e0b', ARRAY['factual_errors', 'citation_fabrication', 'false_reasoning', 'impossible_claims']),
('privacy_violation', 'Privacy Violation', 'Potential privacy concerns and data handling issues', '#ec4899', ARRAY['personal_info_retention', 'cross_session_leakage', 'inference_attacks']),
('misinformation', 'Misinformation', 'Spreading false or misleading information', '#ef4444', ARRAY['conspiracy_theories', 'medical_misinformation', 'political_misinformation', 'scientific_inaccuracy'])
ON CONFLICT (slug) DO NOTHING;

-- ========================================
-- SCHEMA DOCUMENTATION
-- ========================================

/*
 * CAITE Database Schema Documentation
 * ==================================
 * 
 * This schema supports a threat intelligence platform for documenting
 * and analyzing AI chat sessions with a focus on security research.
 * 
 * CORE ENTITIES:
 * --------------
 * 
 * 1. users: Extended user profiles beyond Supabase auth
 *    - Supports researchers, verified experts, and regular users
 *    - Privacy controls and notification preferences
 *    - Verification status for expert privileges
 * 
 * 2. chat_sessions: Core content entities
 *    - Represents a complete chat conversation
 *    - Categorized by security threat type
 *    - Lifecycle: draft -> published -> archived/flagged
 *    - Rich metadata for research and SEO
 * 
 * 3. chat_messages: Individual messages within sessions
 *    - Ordered conversation flow with message_index
 *    - Flexible JSONB content for text, images, attachments
 *    - Metadata stores tool calls, reasoning, etc.
 *    - Annotations for user analysis
 * 
 * CATEGORIZATION SYSTEM:
 * ----------------------
 * 
 * - categories: Predefined security categories (prompt injection, etc.)
 * - tags: User-defined flexible tagging system
 * - session_tags: Many-to-many relationship for flexible categorization
 * - Hierarchical categories with subcategories
 * 
 * SECURITY MODEL:
 * ---------------
 * 
 * - Row Level Security (RLS) enforces data access controls
 * - Published sessions are publicly accessible
 * - Draft sessions only visible to owners
 * - Flagged content restricted to moderators
 * - Expert reviews require verified status
 * - Comprehensive audit trail in security_logs
 * 
 * COMMUNITY FEATURES:
 * -------------------
 * 
 * - user_follows: Research community building
 * - session_comments: Threaded discussions on sessions
 * - expert_reviews: Formal peer review system
 * - session_likes/bookmarks: Content curation
 * - moderation_reports: Community-driven content quality
 * 
 * ANALYTICS & ENGAGEMENT:
 * -----------------------
 * 
 * - session_views: Detailed analytics with device/location tracking
 * - Automatic count updates via triggers
 * - Engagement metrics (likes, shares, bookmarks)
 * - Research-grade analytics for verified researchers
 * 
 * INTEGRATION POINTS:
 * -------------------
 * 
 * - api_tokens: External tool integration
 * - social_shares: Cross-platform content distribution
 * - external_references: Links to papers, CVEs, etc.
 * - Webhook-ready for real-time notifications
 * 
 * PERFORMANCE OPTIMIZATIONS:
 * ---------------------------
 * 
 * - Comprehensive indexing for search and filtering
 * - Full-text search on content and metadata
 * - Partial indexes for common query patterns
 * - Automatic statistics updates via triggers
 * 
 * CONTENT LIFECYCLE:
 * ------------------
 * 
 * 1. Draft Creation:
 *    - User creates session with status='draft'
 *    - Messages added incrementally
 *    - Only visible to owner
 * 
 * 2. Publication:
 *    - Status changed to 'published'
 *    - Content becomes publicly accessible
 *    - SEO metadata activated
 *    - Analytics tracking begins
 * 
 * 3. Community Interaction:
 *    - Comments, likes, bookmarks
 *    - Expert reviews and validations
 *    - Social sharing and citations
 * 
 * 4. Moderation:
 *    - Community reporting system
 *    - Flagging workflow for problematic content
 *    - Expert review for quality assurance
 * 
 * USAGE PATTERNS:
 * ---------------
 * 
 * Common Queries:
 * 
 * -- Get published sessions by category
 * SELECT * FROM chat_sessions 
 * WHERE status = 'published' AND category = 'prompt_injection'
 * ORDER BY published_at DESC;
 * 
 * -- Search sessions by content
 * SELECT s.*, ts_rank(to_tsvector('english', s.title), query) as rank
 * FROM chat_sessions s, plainto_tsquery('english', 'search term') query
 * WHERE to_tsvector('english', s.title) @@ query
 *   AND s.status = 'published'
 * ORDER BY rank DESC;
 * 
 * -- Get user's session analytics
 * SELECT s.title, s.view_count, s.like_count,
 *        COUNT(c.id) as comment_count
 * FROM chat_sessions s
 * LEFT JOIN session_comments c ON s.id = c.session_id
 * WHERE s.user_id = $1 AND s.status = 'published'
 * GROUP BY s.id, s.title, s.view_count, s.like_count;
 * 
 * -- Expert review summary
 * SELECT s.title, AVG(r.technical_accuracy) as avg_accuracy,
 *        COUNT(r.id) as review_count
 * FROM chat_sessions s
 * JOIN expert_reviews r ON s.id = r.session_id
 * WHERE s.category = 'prompt_injection'
 * GROUP BY s.id, s.title
 * HAVING COUNT(r.id) >= 2;
 * 
 * MIGRATION STRATEGY:
 * -------------------
 * 
 * 1. Create tables in dependency order (enums first)
 * 2. Enable RLS on all tables
 * 3. Create policies before inserting data
 * 4. Add triggers and functions
 * 5. Insert seed data
 * 6. Create indexes last for performance
 * 
 * BACKUP CONSIDERATIONS:
 * ----------------------
 * 
 * - Regular backups of chat_sessions and chat_messages (core content)
 * - Analytics data can be regenerated but consider retention
 * - Export capabilities for research data portability
 * - GDPR compliance for user data deletion
 */