-- Script SQL per la creazione delle tabelle di CleanAI in Supabase

-- Tabella users (estende la tabella auth.users di Supabase)
CREATE TABLE public.users (
  id UUID REFERENCES auth.users(id) PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  full_name TEXT,
  role TEXT NOT NULL CHECK (role IN ('admin', 'supervisor', 'operator')),
  phone TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL
);

-- Trigger per inserire automaticamente un record in users quando viene creato un utente in auth.users
CREATE OR REPLACE FUNCTION public.handle_new_user() 
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.users (id, email, role)
  VALUES (NEW.id, NEW.email, 'operator');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE PROCEDURE public.handle_new_user();

-- Tabella clients
CREATE TABLE public.clients (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  address TEXT,
  contact_person TEXT,
  contact_email TEXT,
  contact_phone TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL
);

-- Tabella locations
CREATE TABLE public.locations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  client_id UUID REFERENCES public.clients(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  address TEXT NOT NULL,
  size_sqm NUMERIC,
  floor_type TEXT,
  special_requirements TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL
);

-- Tabella cleaning_tasks
CREATE TABLE public.cleaning_tasks (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  location_id UUID REFERENCES public.locations(id) ON DELETE CASCADE,
  operator_id UUID REFERENCES public.users(id),
  title TEXT NOT NULL,
  description TEXT,
  status TEXT NOT NULL CHECK (status IN ('pending', 'in_progress', 'completed', 'cancelled')),
  priority TEXT NOT NULL CHECK (priority IN ('low', 'medium', 'high', 'urgent')),
  scheduled_at TIMESTAMP WITH TIME ZONE NOT NULL,
  estimated_duration INTEGER, -- in minutes
  completed_at TIMESTAMP WITH TIME ZONE,
  quality_score INTEGER CHECK (quality_score BETWEEN 0 AND 100),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL
);

-- Tabella cleaning_reports
CREATE TABLE public.cleaning_reports (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  task_id UUID REFERENCES public.cleaning_tasks(id) ON DELETE CASCADE,
  operator_id UUID REFERENCES public.users(id),
  before_image_url TEXT,
  after_image_url TEXT,
  quality_score INTEGER CHECK (quality_score BETWEEN 0 AND 100),
  ai_analysis_result JSONB,
  notes TEXT,
  completed_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL
);

-- Tabella iot_sensors
CREATE TABLE public.iot_sensors (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  location_id UUID REFERENCES public.locations(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  type TEXT NOT NULL CHECK (type IN ('humidity', 'temperature', 'motion', 'air_quality', 'occupancy')),
  status TEXT NOT NULL CHECK (status IN ('active', 'inactive', 'maintenance')),
  last_maintenance_at TIMESTAMP WITH TIME ZONE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL
);

-- Tabella sensor_readings
CREATE TABLE public.sensor_readings (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  sensor_id UUID REFERENCES public.iot_sensors(id) ON DELETE CASCADE,
  location_id UUID REFERENCES public.locations(id) ON DELETE CASCADE,
  value NUMERIC NOT NULL,
  unit TEXT NOT NULL,
  timestamp TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL
);

-- Tabella notifications
CREATE TABLE public.notifications (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES public.users(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  message TEXT NOT NULL,
  type TEXT NOT NULL CHECK (type IN ('task_assigned', 'task_reminder', 'task_completed', 'sensor_alert', 'system')),
  read BOOLEAN DEFAULT false,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now() NOT NULL
);

-- Funzione per aggiornare il campo updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Trigger per aggiornare updated_at su tutte le tabelle
CREATE TRIGGER update_users_updated_at
  BEFORE UPDATE ON public.users
  FOR EACH ROW EXECUTE PROCEDURE update_updated_at_column();

CREATE TRIGGER update_clients_updated_at
  BEFORE UPDATE ON public.clients
  FOR EACH ROW EXECUTE PROCEDURE update_updated_at_column();

CREATE TRIGGER update_locations_updated_at
  BEFORE UPDATE ON public.locations
  FOR EACH ROW EXECUTE PROCEDURE update_updated_at_column();

CREATE TRIGGER update_cleaning_tasks_updated_at
  BEFORE UPDATE ON public.cleaning_tasks
  FOR EACH ROW EXECUTE PROCEDURE update_updated_at_column();

CREATE TRIGGER update_iot_sensors_updated_at
  BEFORE UPDATE ON public.iot_sensors
  FOR EACH ROW EXECUTE PROCEDURE update_updated_at_column();

-- Impostazione delle Row Level Security (RLS) policies
ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.clients ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.locations ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.cleaning_tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.cleaning_reports ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.iot_sensors ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.sensor_readings ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.notifications ENABLE ROW LEVEL SECURITY;

-- Policy per users: admin può vedere tutti, gli altri solo se stessi
CREATE POLICY "Admins can see all users" ON public.users
  FOR SELECT USING (auth.jwt() ->> 'role' = 'admin');

CREATE POLICY "Users can see themselves" ON public.users
  FOR SELECT USING (auth.uid() = id);

-- Policy per clients: admin e supervisor possono vedere tutti
CREATE POLICY "Admins and supervisors can manage clients" ON public.clients
  FOR ALL USING (auth.jwt() ->> 'role' IN ('admin', 'supervisor'));

-- Policy per locations: admin e supervisor possono gestire, operatori possono vedere
CREATE POLICY "Admins and supervisors can manage locations" ON public.locations
  FOR ALL USING (auth.jwt() ->> 'role' IN ('admin', 'supervisor'));

CREATE POLICY "Operators can view locations" ON public.locations
  FOR SELECT USING (auth.jwt() ->> 'role' = 'operator');

-- Policy per cleaning_tasks: admin e supervisor possono gestire, operatori vedono i propri
CREATE POLICY "Admins and supervisors can manage tasks" ON public.cleaning_tasks
  FOR ALL USING (auth.jwt() ->> 'role' IN ('admin', 'supervisor'));

CREATE POLICY "Operators can view and update their tasks" ON public.cleaning_tasks
  FOR SELECT USING (auth.jwt() ->> 'role' = 'operator');

CREATE POLICY "Operators can update their tasks" ON public.cleaning_tasks
  FOR UPDATE USING (auth.jwt() ->> 'role' = 'operator' AND operator_id = auth.uid());

-- Policy per notifications: utenti vedono solo le proprie notifiche
CREATE POLICY "Users can see their notifications" ON public.notifications
  FOR SELECT USING (user_id = auth.uid());

CREATE POLICY "Users can update their notifications" ON public.notifications
  FOR UPDATE USING (user_id = auth.uid());
