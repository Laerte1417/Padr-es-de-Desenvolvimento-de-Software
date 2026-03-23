# Padroes-de-Desenvolvimento-de-Software
Laerte de Lara Ferreira Mendes Junior



Diagnóstico de Code Smells

Parte 1 — Identificação dos Code Smells
Foram identificados 6 code smells distintos na classe SistemaBiblioteca, conforme detalhado abaixo.

CS1 — God Class
Localização: Classe SistemaBiblioteca inteira
Por que é um problema: A classe acumula responsabilidades de 5 domínios distintos: gerenciamento de livros, usuários, empréstimos, envio de e-mails e geração de relatórios. Isso viola o Princípio da Responsabilidade Única (SRP) do SOLID, tornando o código difícil de manter, testar e estender.

CS2 — Primitive Obsession
Localização: Atributos livros: any[], usuarios: string[], emprestimos: Date[]
Os dados de livro, usuário e empréstimo são armazenados em arrays posicionais de strings. O acesso por índices numéricos (ex: livro[3], usuario[2]) é completamente sem semântica, propenso a erros de posição e impossível de validar em tempo de compilação com TypeScript.
•	livro[0] = isbn, livro[1] = titulo, livro[2] = autor, livro[3] = status
•	usuario[2] = tipo, usuario[3] = qtd_emprestimos

CS3 — Long Method
Localização: Método realizarEmprestimo()
O método executa ao menos 6 responsabilidades diferentes em sequência: busca de livro por ISBN, busca de usuário por e-mail, validação de disponibilidade, verificação de limite por tipo de usuário, atualização de status, cálculo de data de devolução e envio de e-mail. Cada uma dessas etapas deveria ser um método separado.

CS4 — Switch Statements
Localização: gerarRelatorio(tipo: string) e bloco if/else de tipo de usuário em realizarEmprestimo()
Cadeias extensas de if/else baseadas em valores de string ('PROFESSOR', 'FUNCIONARIO', 'LIVROS', 'USUARIOS') violam o Princípio Aberto/Fechado (OCP). Sempre que um novo tipo for adicionado, o código existente precisa ser modificado, aumentando o risco de regressão.

CS5 — Magic Numbers e Magic Strings
Localização: Toda a classe — métodos realizarEmprestimo(), calcularMulta()
Valores literais hardcoded sem nome nem contexto: o prazo de 14 dias, a multa de R$ 2,50 por dia, os limites de 10 e 5 empréstimos e as strings de status ('DISPONIVEL', 'EMPRESTADO', 'ATIVO') não têm significado claro e precisam ser alterados em vários lugares se a regra de negócio mudar.

CS6 — Inappropriate Intimacy
Localização: Métodos adicionarLivro(), cadastrarUsuario(), realizarEmprestimo()
A lógica de envio de e-mail está diretamente embutida nas operações de negócio. Isso cria um acoplamento forte entre domínios distintos: qualquer mudança na forma de notificar (trocar e-mail por push notification, por exemplo) exige alterar os métodos de negócio.

Parte 2 — Proposta de Refatoração

Refatoração 1 — Extract Class: Livro, Usuario, Emprestimo
Técnica: Extract Class
Substituir os arrays posicionais por classes tipadas com TypeScript, eliminando o Primitive Obsession e dividindo responsabilidades da God Class.

Código antes (problemático):
private livros: any[] = [];
// livro[0]=isbn, livro[1]=titulo, livro[2]=autor, livro[3]=status
private usuarios: string[] = [];
// usuario[0]=nome, usuario[1]=email, usuario[2]=tipo, usuario[3]=qtd

Código depois (refatorado):
// classes/Livro.ts
export type StatusLivro = 'DISPONIVEL' | 'EMPRESTADO';
export class Livro {
  constructor(
    public readonly isbn: string,
    public titulo: string,
    public autor: string,
    public status: StatusLivro = 'DISPONIVEL'
  ) {}
  estaDisponivel(): boolean { return this.status === 'DISPONIVEL'; }
}

// classes/Usuario.ts
export type TipoUsuario = 'PROFESSOR' | 'FUNCIONARIO' | 'ALUNO';
export class Usuario {
  constructor(
    public readonly nome: string,
    public readonly email: string,
    public readonly tipo: TipoUsuario,
    public qtdEmprestimos: number = 0
  ) {}
  limiteEmprestimos(): number {
    const limites = { PROFESSOR: 10, FUNCIONARIO: 5, ALUNO: 3 };
    return limites[this.tipo];
  }
}

Refatoração 2 — Extract Method: realizarEmprestimo()
Técnica: Extract Method
Quebrar o método longo em métodos privados menores, cada um com uma responsabilidade clara.

Código depois (refatorado):
realizarEmprestimo(isbn: string, emailUsuario: string): void {
  const livro = this.buscarLivroPorIsbn(isbn);
  const usuario = this.buscarUsuarioPorEmail(emailUsuario);
  this.validarEmprestimo(livro, usuario);
  this.registrarEmprestimo(livro, usuario);
  this.notificacoes.notificarEmprestimo(usuario.email, livro.titulo);
}

private buscarLivroPorIsbn(isbn: string): Livro {
  const livro = this.livros.find(l => l.isbn === isbn);
  if (!livro) throw new Error('Livro não encontrado');
  return livro;
}

private validarEmprestimo(livro: Livro, usuario: Usuario): void {
  if (!livro.estaDisponivel())
    throw new Error('Livro indisponível');
  if (usuario.qtdEmprestimos >= usuario.limiteEmprestimos())
    throw new Error('Limite de empréstimos atingido');
}

Refatoração 3 — Replace Conditional with Polymorphism: gerarRelatorio()
Técnica: Replace Conditional with Polymorphism / Strategy Pattern
Eliminar a cadeia if/else de gerarRelatorio() usando uma interface e implementações específicas por tipo.

Código depois (refatorado):
// interfaces/RelatorioStrategy.ts
export interface RelatorioStrategy {
  gerar(): string;
}

// relatorios/RelatorioLivros.ts
export class RelatorioLivros implements RelatorioStrategy {
  constructor(private livros: Livro[]) {}
  gerar(): string {
    return this.livros
      .map(l => `${l.isbn} | ${l.titulo} | ${l.autor}`)
      .join('\n');
  }
}

// Em SistemaBiblioteca:
gerarRelatorio(strategy: RelatorioStrategy): string {
  return strategy.gerar();
}

Refatoração 4 — Extract Constant: Magic Numbers e Strings
Técnica: Extract Constant
Substituir literais hardcoded por constantes nomeadas em um arquivo de configuração.

// config/regras.ts
export const REGRAS = {
  PRAZO_DEVOLUCAO_DIAS: 14,
  MULTA_POR_DIA: 2.50,
  LIMITE_PROFESSOR: 10,
  LIMITE_FUNCIONARIO: 5,
  LIMITE_ALUNO: 3,
} as const;

Refatoração 5 — Extract Class: ServicoNotificacao
Técnica: Extract Class / Move Method
Extrair toda a lógica de envio de e-mail para uma classe dedicada, desacoplando notificações das regras de negócio.

// services/ServicoNotificacao.ts
export class ServicoNotificacao {
  notificarEmprestimo(email: string, tituloLivro: string): void {
    console.log(`[EMAIL] Para: ${email} | Livro: ${tituloLivro}`);
  }
  notificarCadastro(email: string, nome: string): void {
    console.log(`[EMAIL] Para: ${email} | Bem-vindo, ${nome}!`);
  }
}

// Em SistemaBiblioteca — injeção de dependência:
class SistemaBiblioteca {
  constructor(
    private notificacoes: ServicoNotificacao = new ServicoNotificacao()
  ) {}
}


Considerações Finais
A refatoração proposta divide a God Class em pelo menos 6 artefatos menores e coesos: as classes de domínio Livro, Usuario e Emprestimo; o serviço ServicoNotificacao; as implementações de RelatorioStrategy; e o arquivo de constantes REGRAS. O resultado é um sistema que respeita os princípios SOLID, é mais fácil de testar unitariamente e permite extensões futuras sem risco de regressão.


