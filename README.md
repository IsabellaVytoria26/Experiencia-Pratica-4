-- SCHEMA: fitmanage

CREATE SCHEMA IF NOT EXISTS fitmanage;
SET search_path = fitmanage;

-- Tabela: plano
CREATE TABLE plano (
  id_plano SERIAL PRIMARY KEY,
  nome_plano VARCHAR(100) NOT NULL,
  valor_mensal NUMERIC(10,2) NOT NULL CHECK (valor_mensal >= 0),
  duracao_meses INTEGER NOT NULL CHECK (duracao_meses > 0)
);

-- Tabela: instrutor
CREATE TABLE instrutor (
  id_instrutor SERIAL PRIMARY KEY,
  nome VARCHAR(150) NOT NULL,
  especialidade VARCHAR(100)
);

-- Tabela: aluno
CREATE TABLE aluno (
  id_aluno SERIAL PRIMARY KEY,
  nome VARCHAR(150) NOT NULL,
  data_nascimento DATE,
  telefone VARCHAR(30),
  email VARCHAR(150),
  data_matricula DATE NOT NULL DEFAULT CURRENT_DATE,
  id_plano INTEGER NOT NULL,
  CONSTRAINT fk_aluno_plano FOREIGN KEY (id_plano)
    REFERENCES plano(id_plano)
    ON UPDATE CASCADE
    ON DELETE RESTRICT,
  CONSTRAINT unq_aluno_email UNIQUE (email)
);

-- Tabela: pagamento
CREATE TABLE pagamento (
  id_pagamento SERIAL PRIMARY KEY,
  id_aluno INTEGER NOT NULL,
  data_pagamento DATE NOT NULL DEFAULT CURRENT_DATE,
  valor NUMERIC(10,2) NOT NULL CHECK (valor >= 0),
  status_pagamento VARCHAR(20) NOT NULL CHECK (status_pagamento IN ('PAGO','PENDENTE','ATRASADO')),
  CONSTRAINT fk_pagamento_aluno FOREIGN KEY (id_aluno)
    REFERENCES aluno(id_aluno)
    ON UPDATE CASCADE
    ON DELETE CASCADE
);

-- Tabela: treino
CREATE TABLE treino (
  id_treino SERIAL PRIMARY KEY,
  id_aluno INTEGER NOT NULL,
  id_instrutor INTEGER NOT NULL,
  nome_treino VARCHAR(120),
  data_criacao DATE NOT NULL DEFAULT CURRENT_DATE,
  observacoes TEXT,
  CONSTRAINT fk_treino_aluno FOREIGN KEY (id_aluno)
    REFERENCES aluno(id_aluno)
    ON UPDATE CASCADE
    ON DELETE CASCADE,
  CONSTRAINT fk_treino_instrutor FOREIGN KEY (id_instrutor)
    REFERENCES instrutor(id_instrutor)
    ON UPDATE CASCADE
    ON DELETE RESTRICT
);

-- Tabela: exercicio
CREATE TABLE exercicio (
  id_exercicio SERIAL PRIMARY KEY,
  nome_exercicio VARCHAR(120) NOT NULL,
  grupo_muscular VARCHAR(80),
  descricao TEXT
);

-- Tabela associativa: treino_exercicio (N:N)
CREATE TABLE treino_exercicio (
  id_treino INTEGER NOT NULL,
  id_exercicio INTEGER NOT NULL,
  posicao INTEGER NOT NULL,            -- ordem no treino
  series INTEGER NOT NULL CHECK (series > 0),
  repeticoes INTEGER NOT NULL CHECK (repeticoes > 0),
  carga NUMERIC(7,2) DEFAULT 0 CHECK (carga >= 0),
  PRIMARY KEY (id_treino, id_exercicio, posicao),
  CONSTRAINT fk_te_ex_treino FOREIGN KEY (id_treino)
    REFERENCES treino(id_treino)
    ON UPDATE CASCADE
    ON DELETE CASCADE,
  CONSTRAINT fk_te_ex_exercicio FOREIGN KEY (id_exercicio)
    REFERENCES exercicio(id_exercicio)
    ON UPDATE CASCADE
    ON DELETE RESTRICT
);

-- Tabela: checkin
CREATE TABLE checkin (
  id_checkin SERIAL PRIMARY KEY,
  id_aluno INTEGER NOT NULL,
  data_hora_checkin TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  meio_checkin VARCHAR(50) DEFAULT 'BALCAO', -- ex: BALCAO, APP, CARTAO
  CONSTRAINT fk_checkin_aluno FOREIGN KEY (id_aluno)
    REFERENCES aluno(id_aluno)
    ON UPDATE CASCADE
    ON DELETE CASCADE
);

-- Índices adicionais recomendados
CREATE INDEX idx_aluno_email ON aluno(email);
CREATE INDEX idx_pagamento_aluno_data ON pagamento (id_aluno, data_pagamento);
CREATE INDEX idx_checkin_aluno_data ON checkin (id_aluno, data_hora_checkin);
CREATE INDEX idx_treino_aluno ON treino(id_aluno);
CREATE INDEX idx_treino_instrutor ON treino(id_instrutor);
-- Trigger function (exemplo simples)
CREATE OR REPLACE FUNCTION atualizar_status_pagamento()
RETURNS TRIGGER AS $$
DECLARE
  v_valor_plano NUMERIC;
BEGIN
  SELECT p.valor_mensal INTO v_valor_plano
  FROM aluno a
  JOIN plano p ON a.id_plano = p.id_plano
  WHERE a.id_aluno = NEW.id_aluno;

  IF NEW.valor >= v_valor_plano THEN
    NEW.status_pagamento := 'PAGO';
  ELSE
    NEW.status_pagamento := 'PENDENTE';
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_pagamento_status
BEFORE INSERT ON pagamento
FOR EACH ROW
EXECUTE FUNCTION atualizar_status_pagamento();
