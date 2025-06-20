# NFL Database Schema Design for PostgreSQL

## Core Tables

### 1. Conferences
```sql
CREATE TABLE conferences (
    conference_id SERIAL PRIMARY KEY,
    conference_name VARCHAR(10) NOT NULL UNIQUE,
    abbreviation CHAR(3) NOT NULL UNIQUE,
    created_date DATE DEFAULT CURRENT_DATE
);
```

### 2. Divisions
```sql
CREATE TABLE divisions (
    division_id SERIAL PRIMARY KEY,
    division_name VARCHAR(20) NOT NULL,
    conference_id INTEGER NOT NULL REFERENCES conferences(conference_id),
    abbreviation VARCHAR(10) NOT NULL UNIQUE,
    UNIQUE(division_name, conference_id)
);
```

### 3. Teams
```sql
CREATE TABLE teams (
    team_id SERIAL PRIMARY KEY,
    team_name VARCHAR(50) NOT NULL,
    city VARCHAR(50) NOT NULL,
    division_id INTEGER NOT NULL REFERENCES divisions(division_id),
    founded_year INTEGER CHECK (founded_year > 1900),
    home_stadium VARCHAR(100),
    abbreviation VARCHAR(3) NOT NULL UNIQUE,
    colors VARCHAR(50),
    head_coach VARCHAR(100),
    owner VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 4. Seasons
```sql
CREATE TABLE seasons (
    season_id SERIAL PRIMARY KEY,
    season_year INTEGER NOT NULL,
    season_type VARCHAR(20) NOT NULL CHECK (season_type IN ('Regular', 'Playoff', 'Preseason')),
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    total_weeks INTEGER DEFAULT 18,
    UNIQUE(season_year, season_type)
);
```

### 5. Players
```sql
CREATE TABLE players (
    player_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    team_id INTEGER REFERENCES teams(team_id),
    position VARCHAR(5) NOT NULL,
    jersey_number INTEGER CHECK (jersey_number BETWEEN 0 AND 99),
    birth_date DATE,
    height_inches INTEGER CHECK (height_inches > 0),
    weight_pounds INTEGER CHECK (weight_pounds > 0),
    college VARCHAR(100),
    draft_year INTEGER,
    draft_round INTEGER,
    draft_pick INTEGER,
    years_pro INTEGER DEFAULT 0,
    status VARCHAR(20) DEFAULT 'Active' CHECK (status IN ('Active', 'Injured', 'Suspended', 'Retired')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(team_id, jersey_number)
);
```

### 6. Games
```sql
CREATE TABLE games (
    game_id SERIAL PRIMARY KEY,
    season_id INTEGER NOT NULL REFERENCES seasons(season_id),
    week INTEGER NOT NULL CHECK (week BETWEEN 1 AND 22),
    game_date TIMESTAMP NOT NULL,
    home_team_id INTEGER NOT NULL REFERENCES teams(team_id),
    away_team_id INTEGER NOT NULL REFERENCES teams(team_id),
    home_score INTEGER DEFAULT 0 CHECK (home_score >= 0),
    away_score INTEGER DEFAULT 0 CHECK (away_score >= 0),
    game_type VARCHAR(20) DEFAULT 'Regular' CHECK (game_type IN ('Regular', 'Playoff', 'Preseason', 'Super Bowl')),
    stadium VARCHAR(100),
    weather_conditions TEXT,
    attendance INTEGER CHECK (attendance >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT different_teams CHECK (home_team_id != away_team_id),
    UNIQUE(season_id, week, home_team_id, away_team_id)
);
```

### 7. Player Statistics (Partitioned by Season)
```sql
-- Parent table for player statistics
CREATE TABLE player_stats (
    stat_id BIGSERIAL,
    player_id INTEGER NOT NULL REFERENCES players(player_id),
    game_id INTEGER NOT NULL REFERENCES games(game_id),
    season_id INTEGER NOT NULL REFERENCES seasons(season_id),
    stat_category VARCHAR(20) NOT NULL,
    stat_type VARCHAR(50) NOT NULL,
    stat_value NUMERIC(10,2) NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (stat_id, season_id)
) PARTITION BY RANGE (season_id);

-- Create partitions for different seasons
CREATE TABLE player_stats_2023 PARTITION OF player_stats
    FOR VALUES FROM (2023) TO (2024);
CREATE TABLE player_stats_2024 PARTITION OF player_stats
    FOR VALUES FROM (2024) TO (2025);
```

### 8. Team Statistics (Partitioned by Season)
```sql
-- Parent table for team statistics
CREATE TABLE team_stats (
    stat_id BIGSERIAL,
    team_id INTEGER NOT NULL REFERENCES teams(team_id),
    game_id INTEGER NOT NULL REFERENCES games(game_id),
    season_id INTEGER NOT NULL REFERENCES seasons(season_id),
    stat_category VARCHAR(20) NOT NULL,
    stat_type VARCHAR(50) NOT NULL,
    stat_value NUMERIC(10,2) NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (stat_id, season_id)
) PARTITION BY RANGE (season_id);

-- Create partitions for different seasons
CREATE TABLE team_stats_2023 PARTITION OF team_stats
    FOR VALUES FROM (2023) TO (2024);
CREATE TABLE team_stats_2024 PARTITION OF team_stats
    FOR VALUES FROM (2024) TO (2025);
```

## Supporting Tables

### 9. Positions
```sql
CREATE TABLE positions (
    position_id SERIAL PRIMARY KEY,
    position_code VARCHAR(5) NOT NULL UNIQUE,
    position_name VARCHAR(50) NOT NULL,
    position_group VARCHAR(20) NOT NULL CHECK (position_group IN ('Offense', 'Defense', 'Special Teams')),
    description TEXT
);
```

### 10. Stadiums
```sql
CREATE TABLE stadiums (
    stadium_id SERIAL PRIMARY KEY,
    stadium_name VARCHAR(100) NOT NULL,
    city VARCHAR(50) NOT NULL,
    state VARCHAR(2) NOT NULL,
    capacity INTEGER CHECK (capacity > 0),
    surface_type VARCHAR(20),
    roof_type VARCHAR(20),
    opened_year INTEGER,
    address TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 11. Plays (For detailed play-by-play data)
```sql
CREATE TABLE plays (
    play_id BIGSERIAL PRIMARY KEY,
    game_id INTEGER NOT NULL REFERENCES games(game_id),
    season_id INTEGER NOT NULL REFERENCES seasons(season_id),
    quarter INTEGER NOT NULL CHECK (quarter BETWEEN 1 AND 4),
    play_clock TIME,
    down INTEGER CHECK (down BETWEEN 1 AND 4),
    yards_to_go INTEGER CHECK (yards_to_go >= 0),
    yard_line INTEGER CHECK (yard_line BETWEEN 0 AND 100),
    play_type VARCHAR(20) NOT NULL,
    play_description TEXT,
    yards_gained INTEGER DEFAULT 0,
    possession_team_id INTEGER REFERENCES teams(team_id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) PARTITION BY RANGE (season_id);
```

## Indexes

### Essential Indexes for Performance

```sql
-- Games table indexes
CREATE INDEX idx_games_date ON games(game_date);
CREATE INDEX idx_games_season_week ON games(season_id, week);
CREATE INDEX idx_games_teams ON games(home_team_id, away_team_id);

-- Players table indexes
CREATE INDEX idx_players_team ON players(team_id);
CREATE INDEX idx_players_position ON players(position);
CREATE INDEX idx_players_name ON players(last_name, first_name);

-- Player stats indexes
CREATE INDEX idx_player_stats_player_game ON player_stats(player_id, game_id);
CREATE INDEX idx_player_stats_category ON player_stats(stat_category, stat_type);
CREATE INDEX idx_player_stats_season ON player_stats(season_id);

-- Team stats indexes
CREATE INDEX idx_team_stats_team_game ON team_stats(team_id, game_id);
CREATE INDEX idx_team_stats_category ON team_stats(stat_category, stat_type);

-- Plays table indexes (for each partition)
CREATE INDEX idx_plays_game ON plays(game_id);
CREATE INDEX idx_plays_type ON plays(play_type);
CREATE INDEX idx_plays_quarter ON plays(quarter);
```

## Constraints and Rules

### Data Integrity Constraints
```sql
-- Ensure jersey numbers are unique per team
ALTER TABLE players ADD CONSTRAINT unique_jersey_per_team 
    UNIQUE(team_id, jersey_number);

-- Ensure game dates are reasonable
ALTER TABLE games ADD CONSTRAINT reasonable_game_date 
    CHECK (game_date >= '1920-01-01' AND game_date <= CURRENT_DATE + INTERVAL '1 year');

-- Ensure statistical values are reasonable
ALTER TABLE player_stats ADD CONSTRAINT reasonable_stat_values 
    CHECK (stat_value >= 0 AND stat_value <= 10000);
```

### Triggers for Data Maintenance
```sql
-- Update player status when traded
CREATE OR REPLACE FUNCTION update_player_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_players_updated
    BEFORE UPDATE ON players
    FOR EACH ROW
    EXECUTE FUNCTION update_player_timestamp();
```

## Views for Common Queries

### Team Roster View
```sql
CREATE VIEW team_rosters AS
SELECT 
    t.team_name,
    t.city,
    p.first_name,
    p.last_name,
    p.position,
    p.jersey_number,
    pos.position_name
FROM teams t
JOIN players p ON t.team_id = p.team_id
JOIN positions pos ON p.position = pos.position_code
WHERE p.status = 'Active'
ORDER BY t.team_name, p.position, p.jersey_number;
```

### Season Statistics View
```sql
CREATE VIEW season_player_stats AS
SELECT 
    p.first_name || ' ' || p.last_name as player_name,
    t.team_name,
    s.season_year,
    ps.stat_category,
    ps.stat_type,
    SUM(ps.stat_value) as total_value,
    COUNT(ps.stat_value) as games_played
FROM players p
JOIN teams t ON p.team_id = t.team_id
JOIN player_stats ps ON p.player_id = ps.player_id
JOIN seasons s ON ps.season_id = s.season_id
GROUP BY p.player_id, p.first_name, p.last_name, t.team_name, s.season_year, ps.stat_category, ps.stat_type
ORDER BY s.season_year, total_value DESC;
```

## Performance Optimization Tips

1. **Partitioning**: Statistics tables are partitioned by season for better query performance and maintenance
2. **Indexing**: Strategic indexes on commonly queried columns (dates, foreign keys, categories)
3. **Data Types**: Use appropriate data types to minimize storage and improve performance
4. **Constraints**: Implement check constraints for data validation at the database level
5. **Normalization**: Properly normalized to 3NF to reduce redundancy while maintaining performance