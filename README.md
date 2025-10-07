# SSTI na prática: identificando a tecnologia por trás do template

SSTI (Server-Side Template Injection) acontece quando o input do usuário é processado diretamente dentro de uma engine de template no servidor.
Isso pode permitir desde simples vazamentos até execução remota de código (RCE).

Após confirmar uma possível SSTI, o próximo desafio é descobrir qual engine de template está processando a entrada — informação essencial para validação, mitigação e reporte. Este guia compartilha a metodologia testada que usamos para descobrir a tecnologia por trás dos templates de forma rápida e confiável.

# <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/nodejs/nodejs-original.svg" width="24" height="24" /> Node.js

### **Handlebars**

#### Indicadores fortes de Handlebars

1. Sintaxe `{{ }}` com blocos/sections usando `{{#if ...}}{{/if}}`, `{{#each ...}}{{/each}}`, `{{#unless ...}}`
2. Comentários: `{{! comment }}` ou `{{!-- comment --}}`
3. Unescaped output com triple moustache: `{{{var}}}`
4. Mensagens de erro/stack trace contendo `handlebars`, `express-handlebars`, `handlebars.runtime`, ou `Missing helper`
5. Presença de helpers custom (servidor) — erros do tipo `Missing helper: "nomeDoHelper"`

#### Testes inócuos

1. Teste de comentário

```
payloads:
  {{!comment_handlebars}}
  {{!-- long comment --}}
```

O que observar:
  - o comentário some do output (é removido) → suporta sintaxe de comentário Handlebars.
  - se o texto `{{!comment_handlebars}}` aparece literal → provavelmente não é Handlebars.

2. Teste de bloco/section (Handlbars tem `#if`, `#each`)

```
payloads:
  {{#if true}}YES{{/if}}
```

O que observar:
  - se aparece `YES` → engine avalia o bloco (Handlbars aceita #if/#each).
  - atenção: outros engines também aceitam blocos (#), então combine com outros testes.

3. Teste de avaliação de expressão (aritmética)

```
payloads:
  {{7*7}}
  {{{7*7}}}
```

O que observar:
  - Handlebars **não** avalia expressões aritméticas por padrão → geralmente retorna `7*7` literal.
  - Se retorna `49` → é Jinja/Twig/Nunjucks/Freemarker etc. (não Handlebars).

4. Teste de unescaped output (triple moustache)

```
payloads:
  {{{"<b>OK</b>"}}}
```

O que observar:
  - se aparece `<b>OK</b>` renderizado sem entidades (tag ativa) → engine suporta `{{{ }}}` (Handlbars e Mustache suportam triple mustache).
  - se aparece `&lt;b&gt;OK&lt;/b&gt;` → escape automático está ativo.

5. Teste de helper/partial indication

```
payloads:
  {{#each [1,2]}}X{{/each}}
  {{> partialName}}
```

O que observar:
  - `each` e `>` (partials) são sintaxes Handlebars; se você vê `X` repetido ou partials sendo incluídos, é um indicativo.
  - se você provoca um erro leve com `{{unknownHelper}}` e resposta contém `Missing helper` → forte sinal de Handlebars.

#### Checklist curto (passo-a-passo)

- Envie `{{7*7}}` → se `49`, descarta Handlebars.
- Envie `{{!hb}}` e `{{!--hb--}}` → se removido, ponto para Handlebars.
- Envie `{{#if true}}OK{{/if}}` → se `OK`, ponto para Handlebars.
- Envie `{{{ "<b>OK</b>" }}}` → se HTML ativo, triple-stash suportado.
- Tente provocar erro leve e cheque por `Missing helper` / `handlebars` no trace.
- Combine evidências; 3+ pontos positivos → muito provável Handlebars.

### **EJS**

#### Indicadores fortes de EJS

1. Presença de delimitadores `<% %>`, `<%= %>`, `<%- %>` nas respostas ou em fontes/paths
2. Arquivos `.ejs` expostos em comentários, erros ou caminhos `(views/*.ejs)`
3. Stack traces/erros Node.js com referências a `ejs` ou `EJS` (ou `express` + referência a arquivos `.ejs`)
4. Headers ou banners: `X-Powered-By: Express` junto com templates que usam `<%` é um bom indício

#### Testes inócuos

1. Teste básico de avaliação (JS expressions)

```
payloads:
  <%= 7*7 %>
```

O que observar:
  - se aparecer `"49"` → engine avalia JS dentro de `<%= %>` (forte indício de EJS ou similar).
  - se aparecer o payload literal (ex.: `<%= 7*7 %>`) → delimitadores não estão sendo processados.

2. Teste de escape vs raw

```
payloads:
  <%= "<b>OK</b>" %>
  <%- "<b>OK</b>" %>
```

O que observar:
  - payload 1 deve aparecer escapado (e.g. `&lt;b&gt;OK&lt;/b&gt;`) se for `<%=`.
  - payload 2 deve aparecer como HTML renderizado (`<b>OK</b>`) se for `<%-`.

3. Teste de scriptlet (não imprime)

```
payloads:
  <% var x = 1; %>HELLO
```

O que observar:
  - scriptlet não imprime nada por si; se `HELLO` aparece sem o scriptlet, ok.
  - se o scriptlet for executado no servidor e afetar saída visível (difícil/raramente detectável), pare — não é necessário.

4. Teste de comentário EJS

- EJS tem comentários via scriptlets como `<% /* comentário */ %>` — injetar isto normalmente não aparece na saída.

5. Provocar erro leve (com cuidado)

```
payloads:
  <%= nonexistentVar %>
```

O que observar:
  - se a resposta/erro traz `ReferenceError: nonexistentVar is not defined` ou stack trace Node, pode revelar Node/EJS.
  - NÃO force crashes — apenas observe mensagens leves se já aparecem.

#### Checklist rápido (passo-a-passo)

- Envie `<%= 7*7 %>` → se `49`, provavelmente engine que avalia JS (EJS / ERB-like; combine com outros testes).
- Envie `<%- "<b>OK</b>" %>` → se HTML renderizado, ponto forte para EJS (raw output).
- Envie `<% /* comment */ %>` → se o comentário some (delimitadores `<%` processados).
- Tente `<%= nonexistentVar %>` e verifique por `ReferenceError`/stack trace Node.
- Procure por `.ejs` em paths, por `X-Powered-By: Express` e por menções a `ejs`/`express` em erros.
- Reúna 3+ evidências antes de concluir.

### **Nunjucks**

#### Indicadores fortes de Nunjucks

1. Sintaxe e delimitadores típicos: `{{ ... }}`, `{% ... %}`, `{# ... #}`.
2. Arquivos `.njk`, `.nunjucks` ou referências a `views/*.njk` expostos em comentários, erros ou caminhos.
3. Stack traces/erros Node.js com referências a `nunjucks`, `nunjucks.configure()` ou `nunjucks.render`.
4. Header `X-Powered-By: Express` combinado com templates que usam `{% %}`/`{{ }}` e presença do pacote `nunjucks` no código/erro.

#### Testes inócuos

1. Teste básico de avaliação (expressões)
```
payloads:
  {{ 7*7 }}
```
O que observar:
  - se aparecer `49` → engine avalia expressões (forte indício de Nunjucks / Jinja-like).
  - se aparecer o payload literal (`{{ 7*7 }}`) → delimitadores não estão sendo processados.

2. Teste de blocos / tags (if / for)
```
payloads:
  {% if true %}YES{% endif %}
  {% for i in [1,2] %}X{% endfor %}
```
O que observar:
  - se aparecer `YES` → `{% if %}` foi interpretado.
  - se aparecer `XX` → `for`/`endfor` processados (sintaxe Jinja-like).

3. Teste de comentário Nunjucks
```
payloads:
  {# teste comentario #}
```
O que observar:
  - se o comentário sumir → sintaxe `{# #}` suportada (indício de engines Jinja-like, incluindo Nunjucks).

4. Teste de filtros (pipes)
```
payloads:
  {{ "abc" | upper }}
  {{ [1,2] | length }}
```
O que observar:
  - se aparecer `ABC` → filtro `upper` aplicado.
  - se aparecer `2` → filtro `length` processado (outro forte indício de Nunjucks).

5. Provocar erro leve (com cuidado)
```
payloads:
  {{ unknownVar }}
```
O que observar:
  - se surgir algo como `unknownVar is not defined` ou stack trace `nunjucks`, confirma o uso.

#### Checklist rápido (passo-a-passo)

- Envie `{{ 7*7 }}` → se `49`, engine JS/Jinja-like (forte indício de Nunjucks).
- Teste `{% if true %}YES{% endif %}` → se `YES`, confirma blocos ativos.
- Use `{{ "abc" | upper }}` → se `ABC`, confirma filtros Nunjucks.
- Verifique se `{# comentario #}` some.
- Observe erros: `nunjucks`, `Template render error`, `nunjucks.configure`, etc.
- Combine 3+ indícios antes de concluir.

### **Pug (ex-Jade)**

#### Indicadores fortes de Pug

1. Sintaxe baseada em indentação e não em delimitadores — ausência de `{% %}`, `{{ }}` ou `<% %>`.
2. Stack traces/erros Node.js com referências a `pug`, `jade`, `pug.compile()`, `pug.render()`.
3. Extensões `.pug` ou `.jade` em caminhos, mensagens de erro ou diretórios (`views/*.pug`).
4. Headers `X-Powered-By: Express` com estruturas HTML altamente minificadas ou indentadas de forma incomum (padrão de saída do Pug).

#### Testes inócuos

1. Teste básico de interpolação
```
payloads:
  #{7*7}
```
O que observar:
  - se aparecer `49` → expressões JS avaliadas dentro de `#{}` (forte indício de Pug).
  - se aparecer o payload literal (`#{7*7}`) → não está sendo processado.

2. Teste de escape vs não escape
```
payloads:
  #{'<b>OK</b>'}
  !{'<b>OK</b>'}
```
O que observar:
  - se o primeiro aparece como texto (`&lt;b&gt;OK&lt;/b&gt;`), é escapado.
  - se o segundo renderiza `<b>OK</b>`, confirma suporte à interpolação raw (`!{}`).

3. Teste de blocos condicionais
```
payloads:
  if true
    OK
```
O que observar:
  - se a saída mostra `OK` → o bloco `if` foi interpretado (forte indício de Pug).

4. Teste de loops
```
payloads:
  each val in [1,2]
    = val
```
O que observar:
  - se aparecer `12` → `each` foi processado, confirmando engine Pug/Jade.

5. Provocar erro leve (com cuidado)
```
payloads:
  #{unknownVar}
```
O que observar:
  - se o erro contiver `pug` ou `jade` (`Cannot read property ... in pug`), confirmação direta.

#### Checklist rápido (passo-a-passo)

- Envie `#{7*7}` → se `49`, indica Pug ou Jade.
- Teste `!{'<b>OK</b>'}` → se renderiza HTML, Pug confirmado.
- Tente bloco `if true\n  OK` → se `OK`, engine processa lógica indentada.
- Procure `.pug` ou `.jade` em erros, paths ou stack traces.
- Mensagens como `pug.compile`, `template.pug` ou `jade.runtime` são confirmação direta.
- Combine 3+ indícios antes de concluir.

### **Mustache**

#### Indicadores fortes de Mustache

1. Sintaxe simples com delimitadores `{{ }}` sem suporte a lógica (sem `{% %}`, `<% %>` ou blocos `if/for`).
2. Erros, stack traces ou comentários com menções a `mustache`, `hogan.js`, `mustache.render()` ou `Hogan.compile()`.
3. Arquivos `.mustache` ou `.hjs` visíveis em caminhos (`views/*.mustache`).
4. Respostas HTML sem interpretação de código JS — apenas substituição de variáveis simples.
5. Presença de delimitadores triplo `{ {{{ }}} }` (não escapados) é um indicativo forte de Mustache/Hogan.

#### Testes inócuos

1. Teste básico de substituição
```
payloads:
  {{7*7}}
```
O que observar:
  - se aparecer literalmente `{{7*7}}` → Mustache **não** avalia expressões (esperado).
  - se aparecer `49` → não é Mustache (provavelmente Nunjucks, Jinja, etc).

2. Teste de variável conhecida
```
payloads:
  {{title}}
```
O que observar:
  - se retornar um valor de contexto (ex.: `"Home"` ou `"Page"`) → substituição de variável simples.
  - se nada ou literal `{{title}}`, variável não está no contexto.

3. Teste de escape vs não escape
```
payloads:
  {{ "<b>OK</b>" }}
  {{{ "<b>OK</b>" }}}
```
O que observar:
  - o primeiro deve aparecer escapado (`&lt;b&gt;OK&lt;/b&gt;`).
  - o segundo renderizado como HTML (`<b>OK</b>`).
  - comportamento típico de Mustache e Hogan.

4. Teste de seção condicional
```
payloads:
  {{#true}}YES{{/true}}
  {{^false}}NO{{/false}}
```
O que observar:
  - Mustache trata seções como blocos de contexto (não executa lógica JS).
  - se aparecer `YESNO` → seções processadas.
  - se literal → não interpretou (provável filtragem ou engine diferente).

5. Teste de comentário Mustache
```
payloads:
  {{! este é um comentário }}
```
O que observar:
  - se o comentário desaparecer da saída → sintaxe `{{! ... }}` suportada (indício claro de Mustache/Hogan).

#### Checklist rápido (passo-a-passo)

- Envie `{{7*7}}` → se **não** vira `49`, engine **não** executa código (característica de Mustache).
- Teste `{{{ "<b>OK</b>" }}}` → se renderiza HTML, confirma suporte a triplo mustache.
- Use `{{! comentario }}` → se o comentário some, confirma Mustache-like.
- Verifique `.mustache` ou `.hjs` em paths, erros ou stack traces.
- Observe mensagens com `mustache.render`, `Hogan.compile`, `Cannot find section`, etc.
- Combine 3+ evidências antes de concluir.

# <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/python/python-original.svg" width="24" height="24" /> Python

### **Jinja2**

#### Indicadores fortes de Jinja2

1. Sintaxe característica com delimitadores `{{ ... }}`, `{% ... %}`, `{# ... #}` (semelhante a Nunjucks).
2. Stack traces ou erros Python com referências a `jinja2`, `TemplateSyntaxError`, `UndefinedError`, ou `jinja2.environment`.
3. Extensões `.jinja`, `.jinja2`, `.html` com menções a Jinja nos caminhos (`templates/*.jinja2`).
4. Cabeçalhos `Server: Werkzeug`, `X-Powered-By: Flask`, ou respostas típicas de frameworks Python.
5. Saída HTML com espaçamento e estrutura idêntica à renderização padrão de Flask/Django templates.

#### Testes inócuos

1. Teste básico de avaliação (expressões)
```
payloads:
  {{ 7*7 }}
```
O que observar:
  - se aparecer `49` → engine avalia expressões (forte indício de Jinja2).
  - se aparecer literal (`{{ 7*7 }}`) → delimitadores não processados.

2. Teste de filtros
```
payloads:
  {{ "abc"|upper }}
  {{ [1,2,3]|length }}
```
O que observar:
  - se aparecer `ABC` → filtro `upper` processado.
  - se aparecer `3` → filtro `length` aplicado (confirmação clara de Jinja2).

3. Teste de blocos / controle de fluxo
```
payloads:
  {% if true %}YES{% endif %}
  {% for i in [1,2] %}X{% endfor %}
```
O que observar:
  - se aparecer `YES` → `{% if %}` interpretado.
  - se aparecer `XX` → laço `for` processado corretamente.

4. Teste de comentários
```
payloads:
  {# comentario #}
```
O que observar:
  - se o comentário sumir → suporte a `{# #}` (forte evidência de Jinja2).

5. Provocar erro leve (com cuidado)
```
payloads:
  {{ unknown_var }}
```
O que observar:
  - se resposta conter `jinja2.exceptions.UndefinedError` ou traceback Python → confirmação direta.

#### Checklist rápido (passo-a-passo)

- Envie `{{ 7*7 }}` → se `49`, engine Jinja-like.
- Teste `{{ "abc"|upper }}` → se `ABC`, confirma filtros Jinja2.
- Use `{% if true %}YES{% endif %}` → se `YES`, confirma blocos ativos.
- Verifique se `{# comentario #}` desaparece.
- Observe erros com `jinja2`, `TemplateSyntaxError`, `UndefinedError`.
- Combine 3+ evidências antes de concluir.

### **Mako**

#### Indicadores fortes de Mako

1. Sintaxe híbrida com delimitadores `<% %>`, `${ }`, `<%! %>` (mistura de estilo JSP e Python).
2. Stack traces ou erros Python com referências a `mako`, `mako.template`, `mako.runtime`, ou `mako.exceptions`.
3. Extensões `.mako` em caminhos, mensagens de erro ou diretórios (`templates/*.mako`).
4. Cabeçalhos `Server: Werkzeug`, `Server: gunicorn`, ou aplicações Python (Flask/Pyramid) com saída de template.
5. Código HTML misturado com tags `<% ... %>` ou `${ ... }` é sinal clássico de Mako.

#### Testes inócuos

1. Teste básico de avaliação
```
payloads:
  ${7*7}
```
O que observar:
  - se aparecer `49` → expressões Python avaliadas (forte indício de Mako).
  - se aparecer literal `${7*7}` → delimitadores não processados.

2. Teste de bloco de código
```
payloads:
  <% x = 1 %>OK
```
O que observar:
  - se `OK` for exibido sem erro → bloco `<% %>` aceito (característico de Mako).
  - se erro ou literal aparecer → engine diferente (EJS, Jinja, etc.).

3. Teste de controle de fluxo
```
payloads:
  <% if True: %>YES<% endif %>
```
O que observar:
  - se `YES` for exibido → confirma execução de blocos Python no servidor.
  - se literal → delimitadores não processados.

4. Teste de escape vs não escape
```
payloads:
  ${"<b>OK</b>"}
```
O que observar:
  - por padrão, Mako **não escapa** HTML (renderiza `<b>OK</b>`).
  - se escapado (`&lt;b&gt;OK&lt;/b&gt;`), engine pode estar configurada com `default_filters=['h']`.

5. Provocar erro leve (com cuidado)
```
payloads:
  ${unknown_var}
```
O que observar:
  - se surgir erro `NameError: name 'unknown_var' is not defined` ou stack trace `mako.exceptions` → confirmação direta.

#### Checklist rápido (passo-a-passo)

- Envie `${7*7}` → se `49`, expressões Python ativas (forte indício de Mako).
- Teste `<% if True: %>YES<% endif %>` → se `YES`, blocos de controle processados.
- Verifique se `${"<b>OK</b>"}` renderiza HTML direto (sem escapar).
- Observe erros: `mako.exceptions`, `mako.template`, `NameError`.
- Procure `.mako` em caminhos, mensagens ou stack traces.
- Combine 3+ evidências antes de concluir.

### **Django Templates**

#### Indicadores fortes de Django Template Engine

1. Sintaxe característica com delimitadores `{{ ... }}` e `{% ... %}`, porém **sem suporte a execução direta de código Python**.
2. Stack traces/erros Python com menções a `django.template`, `TemplateSyntaxError`, `VariableDoesNotExist`, ou `django.core`.
3. Extensões `.html` em diretórios `templates/` (padrão do Django).
4. Cabeçalhos `Server: WSGIServer`, `X-Frame-Options: DENY`, ou páginas de erro com o estilo visual padrão do Django (`<h1>TemplateSyntaxError</h1>`).
5. Presença de filtros e tags específicas do Django (`|safe`, `|length`, `|upper`, `{% url %}`, `{% csrf_token %}`, etc.).

#### Testes inócuos

1. Teste básico de substituição
```
payloads:
  {{ 7*7 }}
```
O que observar:
  - se o resultado for literal `{{ 7*7 }}` → **não** executa código (esperado no Django).
  - se `49` aparecer → **não é Django** (provável Jinja2, Nunjucks, etc.).

2. Teste de variável conhecida
```
payloads:
  {{ request.path }}
  {{ user.username }}
```
O que observar:
  - se a aplicação retornar valores válidos (ex.: `/index/` ou `admin`) → variável de contexto resolvida.
  - se nada for exibido → variável inexistente ou contexto limitado.

3. Teste de filtro
```
payloads:
  {{ "abc"|upper }}
  {{ [1,2,3]|length }}
```
O que observar:
  - se aparecer `ABC` e `3` → filtros padrão de Django funcionando.
  - se erro de sintaxe aparecer (`Invalid filter`) → possivelmente engine customizada.

4. Teste de bloco condicional
```
payloads:
  {% if True %}YES{% endif %}
```
O que observar:
  - se erro de sintaxe ocorrer (`Invalid block tag`) → Django não reconhece `True` literal (espera variável de contexto).
  - se `YES` aparecer → engine Jinja-like, **não** Django.

5. Teste de comentário
```
payloads:
  {% comment %}teste{% endcomment %}
```
O que observar:
  - se o comentário desaparecer → engine Django.
  - se literal

# <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/php/php-original.svg" width="24" height="24" /> PHP

### **Twig**

#### Indicadores fortes de Twig

1. Sintaxe característica com delimitadores `{{ ... }}`, `{% ... %}`, `{# ... #}` (Jinja-like, mas em PHP).
2. Arquivos `.twig` ou menções a `templates/*.twig` em erros ou caminhos.
3. Stack traces PHP ou mensagens de erro com referências a `Twig\Environment`, `Twig\Error`, `Twig\Loader`.
4. Cabeçalhos típicos de aplicações PHP (`X-Powered-By: PHP`, `Server: nginx/apache`) combinados com templates Twig.
5. Filtros e funções Twig específicas (`|escape`, `|length`, `|upper`, `|raw`, `|join`) presentes nas respostas.

#### Testes inócuos

1. Teste básico de avaliação
```
payloads:
  {{ 7*7 }}
```
O que observar:
  - se aparecer `49` → expressões avaliadas (forte indício de Twig).
  - se aparecer literal `{{ 7*7 }}` → delimitadores não processados.

2. Teste de filtro
```
payloads:
  {{ "abc"|upper }}
  {{ [1,2,3]|length }}
```
O que observar:
  - `ABC` → filtro `upper` processado.
  - `3` → filtro `length` processado (confirmação Twig).

3. Teste de bloco condicional
```
payloads:
  {% if true %}YES{% endif %}
```
O que observar:
  - se aparecer `YES` → bloco `{% %}` interpretado.
  - se literal → delimitadores não processados.

4. Teste de loop
```
payloads:
  {% for i in [1,2] %}X{% endfor %}
```
O que observar:
  - se aparecer `XX` → loop processado, confirma engine Twig.

5. Teste de comentário
```
payloads:
  {# comentário #}
```
O que observar:
  - se o comentário sumir → suporte a `{# #}` (indício claro de Twig).

6. Provocar erro leve (com cuidado)
```
payloads:
  {{ unknownVar }}
```
O que observar:
  - se mensagem de erro contém `Twig\Error\RuntimeError` ou `Variable "unknownVar" does not exist` → confirmação direta.

#### Checklist rápido (passo-a-passo)

- Envie `{{ 7*7 }}` → se `49`, engine Twig ativa.
- Teste `{{ "abc"|upper }}` → se `ABC`, filtros Twig funcionando.
- Use `{% if true %}YES{% endif %}` → se `YES`, blocos interpretados.
- Verifique `{# comentário #}` → se desaparece, confirma engine.
- Observe erros: `Twig\Environment`, `Twig\Error`, `Variable "..." does not exist`.
- Procure `.twig` em paths ou mensagens de erro.
- Combine 3+ evidências antes de concluir.

### **Smarty**

#### Indicadores fortes de Smarty

1. Sintaxe típica com delimitadores `{...}` para variáveis e funções, `{* ... *}` para comentários.
2. Arquivos `.tpl` visíveis em caminhos (`templates/*.tpl`) ou mensagens de erro.
3. Stack traces PHP ou mensagens de erro com referências a `Smarty_Internal_Template`, `SmartyException`, `Smarty->fetch()`.
4. Cabeçalhos PHP típicos (`X-Powered-By: PHP`) combinados com templates com sintaxe `{variable}`.
5. Funções e modificadores Smarty (`{$var|escape}`, `{$arr|count}`, `{foreach $items as $i}`) presentes nas respostas.

#### Testes inócuos

1. Teste básico de variável
```
payloads:
  {$testVar}
```
O que observar:
  - se a variável é substituída pelo valor do contexto → engine Smarty processando.
  - se aparece literal `{$testVar}` → delimitadores não processados.

2. Teste de escape / raw
```
payloads:
  {$htmlVar}
  {$htmlVar nofilter}
```
O que observar:
  - primeiro renderiza escapado (`&lt;b&gt;OK&lt;/b&gt;`).
  - segundo renderiza HTML literal (`<b>OK</b>`) → suporte a `nofilter`.

3. Teste de loops
```
payloads:
  {foreach $items as $item}
    {$item}
  {/foreach}
```
O que observar:
  - se itens aparecem concatenados → loop processado (confirma Smarty).

4. Teste de condição
```
payloads:
  {if $cond}YES{/if}
```
O que observar:
  - se `YES` aparece → bloco condicional processado.
  - se literal → delimitadores não processados.

5. Teste de comentário
```
payloads:
  {* comentário *}
```
O que observar:
  - comentário sumindo da saída → suporte a `{* *}`.

6. Provocar erro leve (com cuidado)
```
payloads:
  {$unknownVar}
```
O que observar:
  - se erro ou warning PHP com referência a `Smarty` → confirmação da engine.

#### Checklist rápido (passo-a-passo)

- Envie `{$testVar}` → se variável é resolvida, Smarty ativa.
- Teste `{$htmlVar nofilter}` → se renderiza HTML, confirma suporte a raw output.
- Use `{foreach $items as $item}...{/foreach}` → se loop processa, engine confirmada.
- Teste `{* comentário *}` → se desaparece, engine Smarty.
- Observe erros/stack traces com `Smarty_Internal_Template`, `SmartyException`.
- Procure `.tpl` em paths ou mensagens de erro.
- Combine 3+ evidências antes de concluir.

### **Blade**

#### Indicadores fortes de Blade

1. Sintaxe característica com delimitadores `{{ ... }}` para variáveis e `{!! ... !!}` para saída sem escape.
2. Arquivos `.blade.php` em diretórios `resources/views/*.blade.php`.
3. Stack traces ou erros PHP com referências a `Illuminate\View\View`, `BladeCompiler`, ou `ViewException`.
4. Cabeçalhos PHP típicos (`X-Powered-By: PHP`) combinados com templates Blade.
5. Diretivas Blade específicas: `@if`, `@foreach`, `@include`, `@csrf`, `@extends`, `@section`, etc.

#### Testes inócuos

1. Teste básico de variável
```
payloads:
  {{ 7*7 }}
```
O que observar:
  - se aparecer `49` → expressão é avaliada (Blade processa PHP/expressões).
  - se aparecer literal `{{ 7*7 }}` → delimitadores não processados.

2. Teste de escape vs raw
```
payloads:
  {{ "<b>OK</b>" }}
  {!! "<b>OK</b>" !!}
```
O que observar:
  - primeiro deve aparecer escapado (`&lt;b&gt;OK&lt;/b&gt;`).
  - segundo deve aparecer como HTML renderizado (`<b>OK</b>`).

3. Teste de diretiva condicional
```
payloads:
  @if(true)
    YES
  @endif
```
O que observar:
  - se `YES` aparece → diretiva interpretada (confirma Blade).
  - se literal → não processada.

4. Teste de loop
```
payloads:
  @foreach([1,2] as $i)
    {{ $i }}
  @endforeach
```
O que observar:
  - se `12` aparece → loop processado.

5. Teste de comentário
```
payloads:
  {{-- comentário --}}
```
O que observar:
  - comentário sumindo da saída → suporte a Blade.

6. Provocar erro leve (com cuidado)
```
payloads:
  {{ $unknownVar }}
```
O que observar:
  - se erro ou stack trace PHP referir `BladeCompiler`, `ViewException` → confirmação da engine.

#### Checklist rápido (passo-a-passo)

- Envie `{{ 7*7 }}` → se `49`, Blade processando expressões.
- Teste `{!! "<b>OK</b>" !!}` → HTML renderizado sem escape, confirma raw output.
- Use `@if(true)...@endif` e `@foreach` → se processados, diretivas Blade ativas.
- Teste comentário `{{-- comentário --}}` → se desaparece, engine confirmada.
- Observe erros/stack traces com `BladeCompiler`, `ViewException`, `Illuminate\View\View`.
- Procure `.blade.php` em paths ou mensagens de erro.
- Combine 3+ evidências antes de concluir.

# <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/java/java-original.svg" width="24" height="24" />  Java

### **FreeMarker**

#### Indicadores fortes de FreeMarker

1. Sintaxe característica com `${ ... }` para variáveis e `<# ... >` ou `<#-- ... -->` para diretivas e comentários.
2. Arquivos `.ftl` visíveis em caminhos (`templates/*.ftl`) ou mensagens de erro.
3. Stack traces Java ou mensagens de erro com referências a `freemarker.template.Template`, `freemarker.core`, `TemplateException`.
4. Cabeçalhos típicos de servidores Java (`Server: Apache-Coyote/1.1`, `Servlet`, `Spring`) combinados com templates FreeMarker.
5. Diretivas FreeMarker específicas: `<#if>`, `<#list>`, `<#include>`, `<#macro>`, `<#assign>`.

#### Testes inócuos

1. Teste básico de variável
```
payloads:
  ${7*7}
```
O que observar:
  - se aparecer `49` → expressão avaliada (FreeMarker processando).
  - se aparecer literal `${7*7}` → delimitadores não processados.

2. Teste de diretiva condicional
```
payloads:
  <#if true>
    YES
  </#if>
```
O que observar:
  - se `YES` aparece → bloco `<#if>` interpretado.
  - se literal → não processado.

3. Teste de loop
```
payloads:
  <#list [1,2] as i>
    ${i}
  </#list>
```
O que observar:
  - se `12` aparece → loop processado (confirma FreeMarker).

4. Teste de comentário
```
payloads:
  <#-- comentário -->
```
O que observar:
  - comentário sumindo da saída → suporte a FreeMarker.

5. Teste de escape / raw
```
payloads:
  ${"<b>OK</b>"?html}
```
O que observar:
  - saída escapada como `&lt;b&gt;OK&lt;/b&gt;` → demonstra filtro `?html`.
  - se usar `${"<b>OK</b>"?no_esc}` (ou equivalente) → saída sem escape.

6. Provocar erro leve (com cuidado)
```
payloads:
  ${unknownVar}
```
O que observar:
  - se mensagem de erro Java indicar `freemarker.core.InvalidReferenceException` → confirmação direta.

#### Checklist rápido (passo-a-passo)

- Envie `${7*7}` → se `49`, engine FreeMarker processando expressões.
- Teste `<#if true>YES</#if>` → se `YES`, diretivas FreeMarker interpretadas.
- Teste `<#list [1,2] as i>${i}</#list>` → se `12`, loops processados.
- Verifique comentários `<#-- ... -->` → se desaparecem, engine confirmada.
- Observe erros/stack traces com `freemarker.template`, `freemarker.core`, `InvalidReferenceException`.
- Procure `.ftl` em paths ou mensagens de erro.
- Combine 3+ evidências antes de concluir.

### **Apache Velocity**

#### Indicadores fortes de Velocity

1. Sintaxe característica com `$variable` para variáveis e `#directive(...)` para diretivas (`#if`, `#foreach`, `#set`, `#include`).
2. Arquivos `.vm` visíveis em caminhos (`templates/*.vm`) ou mensagens de erro.
3. Stack traces Java ou mensagens de erro com referências a `org.apache.velocity.Template`, `VelocityEngine`, `VelocityException`.
4. Cabeçalhos típicos de servidores Java (`Server: Apache-Coyote/1.1`, `Servlet`) combinados com templates Velocity.
5. Diretivas específicas: `#if`, `#foreach`, `#set`, `#include`, `#macro`.

#### Testes inócuos

1. Teste básico de variável
```
payloads:
  $testVar
```
O que observar:
  - se substituído pelo valor de contexto → engine Velocity processando.
  - se literal `$testVar` → delimitadores não processados.

2. Teste de diretiva condicional
```
payloads:
  #if($true)
    YES
  #end
```
O que observar:
  - se `YES` aparece → diretiva condicional interpretada.
  - se literal → não processado.

3. Teste de loop
```
payloads:
  #foreach($i in [1,2])
    $i
  #end
```
O que observar:
  - se `12` aparece → loop processado (confirma Velocity).

4. Teste de comentário
```
payloads:
  ## comentário
```
O que observar:
  - comentário sumindo da saída → suporte a `##` ou `#* ... *#`.

5. Provocar erro leve (com cuidado)
```
payloads:
  $unknownVar
```
O que observar:
  - se mensagem de erro Java indicar `VelocityException` ou referência a `unknownVar` → confirmação direta.

#### Checklist rápido (passo-a-passo)

- Envie `$testVar` → se variável é resolvida, Velocity ativa.
- Teste `#if($true)YES#end` → se `YES`, diretiva interpretada.
- Teste `#foreach($i in [1,2])$i#end` → se `12`, loops processados.
- Verifique comentários `## comentário` → se desaparecem, engine confirmada.
- Observe erros/stack traces com `VelocityEngine`, `VelocityException`, `org.apache.velocity.Template`.
- Procure `.vm` em paths ou mensagens de erro.
- Combine 3+ evidências antes de concluir.

### **Thymeleaf**

#### Indicadores fortes de Thymeleaf

1. Sintaxe característica com atributos HTML `th:text`, `th:utext`, `th:if`, `th:each`, `th:include`, `th:replace`.
2. Arquivos `.html` em diretórios `templates/` com namespace de atributos `th:`.
3. Stack traces ou erros Java com referências a `org.thymeleaf.TemplateEngine`, `TemplateProcessingException`.
4. Cabeçalhos típicos de servidores Java (`Server: Apache-Coyote/1.1`, `Servlet`) combinados com templates Thymeleaf.
5. Uso de expressões `${...}` dentro de atributos `th:` para interpolação de variáveis.

#### Testes inócuos

1. Teste básico de variável
```
payloads:
  <p th:text="${7*7}">Fallback</p>
```
O que observar:
  - se aparece `49` → expressão avaliada (Thymeleaf processando).
  - se permanece `Fallback` → delimitadores não processados.

2. Teste de escape vs raw
```
payloads:
  <p th:text="'<b>OK</b>'">Fallback</p>
  <p th:utext="'<b>OK</b>'">Fallback</p>
```
O que observar:
  - `th:text` → saída escapada (`&lt;b&gt;OK&lt;/b&gt;`).
  - `th:utext` → saída renderizada como HTML (`<b>OK</b>`).

3. Teste de condição
```
payloads:
  <p th:if="${true}">YES</p>
```
O que observar:
  - se `YES` aparece → diretiva `th:if` interpretada.
  - se `YES` não aparece → não processado.

4. Teste de loop
```
payloads:
  <ul>
    <li th:each="i : ${[1,2]}">${i}</li>
  </ul>
```
O que observar:
  - se aparece `1 2` → loop processado (Thymeleaf ativo).

5. Provocar erro leve (com cuidado)
```
payloads:
  <p th:text="${unknownVar}">Fallback</p>
```
O que observar:
  - se mensagem de erro Java indicar `TemplateProcessingException` → confirmação direta.

#### Checklist rápido (passo-a-passo)

- Teste `<p th:text="${7*7}">Fallback</p>` → se `49`, engine Thymeleaf processando expressões.
- Use `<p th:utext="'<b>OK</b>'">Fallback</p>` → saída HTML, raw output.
- Teste `<p th:if="${true}">YES</p>` → se `YES`, diretiva interpretada.
- Teste loop `<li th:each="i : ${[1,2]}">${i}</li>` → se `1 2`, iteração processada.
- Observe erros/stack traces com `TemplateProcessingException`, `org.thymeleaf.TemplateEngine`.
- Procure `.html` com atributos `th:` em paths ou mensagens de erro.
- Combine 3+ evidências antes de concluir.

# <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/ruby/ruby-original.svg" width="24" height="24" /> Ruby

### **ERB**

#### Indicadores fortes de ERB

1. Sintaxe característica com delimitadores `<% ... %>` para execução de código e `<%= ... %>` para output.
2. Arquivos `.erb` ou `.html.erb` em diretórios `app/views/*.erb` ou `app/views/**/*.html.erb`.
3. Stack traces ou erros Ruby com referências a `ERB::Compiler`, `ERB#result`, `ActionView::Template::Error`.
4. Cabeçalhos típicos de servidores Ruby (`Server: Puma`, `Server: WEBrick`) combinados com templates ERB.
5. Uso de `<%# ... %>` para comentários, que não aparecem na saída renderizada.

#### Testes inócuos

1. Teste básico de avaliação (expressões)
```
payloads:
  <%= 7*7 %>
```
O que observar:
  - se aparecer `49` → ERB processando expressão Ruby.
  - se aparecer literal `<%= 7*7 %>` → delimitadores não processados.

2. Teste de código sem output (scriptlet)
```
payloads:
  <% x = 1 %>HELLO
```
O que observar:
  - scriptlet não imprime nada; `HELLO` deve aparecer normalmente.
  - se scriptlet gera saída inesperada → comportamento anômalo.

3. Teste de escape vs raw
```
payloads:
  <%= "<b>OK</b>" %>
```
O que observar:
  - saída padrão renderiza `<b>OK</b>` como HTML, ERB não escapa por padrão.

4. Teste de comentário
```
payloads:
  <%# este é um comentário %>
```
O que observar:
  - comentário não aparece na saída renderizada.

5. Provocar erro leve (com cuidado)
```
payloads:
  <%= unknown_var %>
```
O que observar:
  - se erro Ruby do tipo `NameError` ou stack trace `ERB` aparece → confirmação da engine.

#### Checklist rápido (passo-a-passo)

- Envie `<%= 7*7 %>` → se `49`, ERB processando expressão.
- Teste `<% x = 1 %>HELLO` → se `HELLO` aparece, scriptlets aceitos.
- Use `<%= "<b>OK</b>" %>` → HTML renderizado sem escape.
- Teste comentário `<%# comentário %>` → se sumir, confirma ERB.
- Observe erros/stack traces com `ERB::Compiler`, `ERB#result`, `NameError`.
- Procure `.erb` ou `.html.erb` em paths ou mensagens de erro.
- Combine 3+ evidências antes de concluir.

### **Liquid**

#### Indicadores fortes de Liquid

1. Sintaxe característica com delimitadores `{{ ... }}` para variáveis e `{% ... %}` para tags.
2. Arquivos `.liquid` ou `.html` com tags Liquid em diretórios de templates (`templates/*.liquid`).
3. Stack traces ou erros Ruby com referências a `Liquid::Template`, `Liquid::Error`, `Liquid::ParseError`.
4. Uso de filtros Liquid (`| upcase`, `| size`, `| strip`, `| escape`) e tags (`for`, `if`, `include`, `assign`).
5. Comentários `{% comment %} ... {% endcomment %}` ou `{# ... #}`.

#### Testes inócuos

1. Teste básico de variável
```
payloads:
  {{ 7*7 }}
```
O que observar:
  - Liquid **não avalia expressões Ruby** → deve aparecer literal `{{ 7*7 }}`.
  - Se `49` aparece → não é Liquid (provável ERB/Jinja).

2. Teste de variável conhecida
```
payloads:
  {{ page.title }}
```
O que observar:
  - se valor do contexto aparece → substituição de variável simples.
  - se literal → variável não definida.

3. Teste de filtro
```
payloads:
  {{ "abc" | upcase }}
  {{ [1,2,3] | size }}
```
O que observar:
  - `ABC` → filtro aplicado.
  - `3` → filtro aplicado (indício de Liquid).

4. Teste de bloco condicional
```
payloads:
  {% if true %}YES{% endif %}
```
O que observar:
  - se `YES` aparece → bloco processado.
  - se literal → não processado.

5. Teste de loop
```
payloads:
  {% for i in (1..2) %}{{ i }}{% endfor %}
```
O que observar:
  - se `12` aparece → loop processado (confirma Liquid).

6. Teste de comentário
```
payloads:
  {% comment %} comentário {% endcomment %}
```
O que observar:
  - comentário sumindo da saída → suporte a Liquid.

#### Checklist rápido (passo-a-passo)

- Envie `{{ 7*7 }}` → se literal, Liquid ativo (não executa código).
- Teste filtros `{{ "abc" | upcase }}` → se `ABC`, confirma suporte a filtros.
- Use `{% if true %}YES{% endif %}` → se processado, blocos condicional ativos.
- Teste loops `{% for i in (1..2) %}{{ i }}{% endfor %}` → se `12`, iteração processada.
- Teste comentário `{% comment %} ... {% endcomment %}` → se sumir, engine confirmada.
- Observe erros/stack traces com `Liquid::Template`, `Liquid::Error`, `Liquid::ParseError`.
- Combine 3+ evidências antes de concluir.

# <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/go/go-original.svg" width="24" height="24" />  Go

### **Go Templates — text/template & html/template**

#### Indicadores fortes de Go Templates

1. Sintaxe característica com delimitadores `{{ ... }}` para variáveis, funções e blocos de controle.
2. Arquivos `.tmpl`, `.gohtml`, `.html` ou diretórios `templates/` usados em projetos Go.
3. Stack traces ou erros Go com referências a `text/template.Template`, `html/template.Template`, `Execute`, `ExecuteTemplate`.
4. Uso de funções internas de Go Templates (`len`, `index`, `printf`, `html`, `urlquery`).
5. Controle de fluxo: `{{ if ... }}`, `{{ range ... }}`, `{{ with ... }}`, `{{ end }}`.

#### Testes inócuos

1. Teste básico de variável
```
payloads:
  {{ 7*7 }}
```
O que observar:
  - Go Templates **não avalia expressões aritméticas literais** → deve aparecer literal `{{ 7*7 }}`.
  - Se `49` aparece → engine diferente (Jinja, ERB, etc.).

2. Teste de variável conhecida
```
payloads:
  {{ .Title }}
```
O que observar:
  - se valor do contexto aparece → substituição de variável simples.
  - se literal → variável não definida.

3. Teste de bloco condicional
```
payloads:
  {{ if true }}YES{{ end }}
```
O que observar:
  - se `YES` aparece → bloco processado (Go Template ativo).
  - se literal → delimitadores não processados.

4. Teste de loop / range
```
payloads:
  {{ range $i, $v := .Items }}{{ $v }}{{ end }}
```
O que observar:
  - se valores do slice/array aparecem concatenados → loop processado.

5. Teste de comentário
```
payloads:
  {{/* comentário */}}
```
O que observar:
  - comentário sumindo da saída → suporte a Go Templates.

6. Provocar erro leve (com cuidado)
```
payloads:
  {{ .UnknownVar }}
```
O que observar:
  - se mensagem de erro do Go indicar `template: ...: can't evaluate field UnknownVar` → confirmação direta.

#### Checklist rápido (passo-a-passo)

- Envie `{{ 7*7 }}` → se literal, Go Template ativo (não executa aritmética).
- Teste `{{ .Title }}` → se valor do contexto aparece, substituição de variável funcionando.
- Teste blocos `{{ if true }}YES{{ end }}` → se `YES`, diretivas processadas.
- Teste loops `{{ range $i, $v := .Items }}{{ $v }}{{ end }}` → se processado, range ativo.
- Teste comentário `{{/* comentário */}}` → se sumir, engine confirmada.
- Observe erros/stack traces com `text/template.Template` ou `html/template.Template`.
- Combine 3+ evidências antes de concluir.

# <img src="https://cdn.jsdelivr.net/gh/devicons/devicon/icons/dot-net/dot-net-original.svg" width="24" height="24" /> .NET

### **Razor**

#### Indicadores fortes de Razor

1. Sintaxe característica com `@` para variáveis, expressões e blocos de código (`@{ ... }`).
2. Arquivos `.cshtml` ou `.vbhtml` em diretórios `Views/` de projetos ASP.NET ou ASP.NET Core.
3. Stack traces ou erros .NET com referências a `Microsoft.AspNetCore.Mvc.Razor.RazorViewEngine`, `RazorPage`, ou `RazorCompilation`.
4. Cabeçalhos típicos de servidores ASP.NET (`Server: Kestrel`, `Server: IIS`) combinados com templates Razor.
5. Diretivas Razor específicas: `@if`, `@for`, `@foreach`, `@section`, `@Html.Partial`, `@RenderBody`, `@model`.

#### Testes inócuos

1. Teste básico de variável / expressão
```
payloads:
  @DateTime.Now.Year
```
O que observar:
  - se aparece o ano atual → Razor processando expressão C#.
  - se literal `@DateTime.Now.Year` → delimitadores não processados.

2. Teste de bloco de código
```
payloads:
  @{
      var x = 1;
  }
  HELLO
```
O que observar:
  - scriptlet não imprime nada por si; `HELLO` deve aparecer normalmente.
  - se valor do código aparece → expressão processada.

3. Teste de diretiva condicional
```
payloads:
  @if(true){
      <text>YES</text>
  }
```
O que observar:
  - se `YES` aparece → diretiva interpretada (confirma Razor).
  - se literal → não processado.

4. Teste de loop
```
payloads:
  @for(int i = 1; i <= 2; i++){
      @i
  }
```
O que observar:
  - se `12` aparece → loop processado corretamente.

5. Teste de comentário
```
payloads:
  @* comentário *@
```
O que observar:
  - comentário sumindo da saída → suporte a Razor.

6. Provocar erro leve (com cuidado)
```
payloads:
  @unknownVar
```
O que observar:
  - se erro C# ou stack trace mencionar `RazorPage` ou `RazorCompilation` → confirmação direta.

#### Checklist rápido (passo-a-passo)

- Envie `@DateTime.Now.Year` → se valor aparece, Razor processando expressões C#.
- Teste bloco `@{ var x=1; }HELLO` → se `HELLO`, scriptlets aceitos.
- Use `@if(true){<text>YES</text>}` → se processado, diretiva interpretada.
- Teste loop `@for(int i=1;i<=2;i++){@i}` → se `12`, iteração ativa.
- Teste comentário `@* comentário *@` → se sumir, engine confirmada.
- Observe erros/stack traces com `RazorPage`, `RazorCompilation`, `Microsoft.AspNetCore.Mvc.Razor`.
- Combine 3+ evidências antes de concluir.
