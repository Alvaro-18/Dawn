
# Teste Técnico – Desenvolvedor(a) Front-end (Shopify) - check

## Informações gerais

**Data de início:** 29/01/2026

**Data de término:** 30/01/2026

**Tempo total gasto:** 3h30m

---

## Tarefas técnicas

### Exibir vídeo na seção “Imagem com texto” (Image with text) – 32 minutos

### Requisitos

1. O vídeo deve ser editável via personalizador de tema (Theme Customizer).
2. O vídeo deve ser hospedado no Shopify.
3. O vídeo deve ser reproduzido automaticamente (autoplay).
4. O vídeo deve ser carregado utilizando lazy loading.

### Passos do desenvolvimento

1. A section existente foi adicionada ao tema para análise das personalizações disponíveis no Theme Customizer.
    
    Essa etapa foi necessária para garantir que todas as funcionalidades existentes fossem preservadas, independentemente do conteúdo exibido ser uma imagem ou um vídeo.
    
2. Considerando que a section passou a suportar tanto imagem quanto vídeo, uma boa prática seria renomear o arquivo e a section para refletir essa dupla funcionalidade, facilitando o entendimento tanto para desenvolvedores quanto para o cliente final.
    
    No entanto, por se tratar de um teste técnico, o nome original do arquivo foi mantido para facilitar a revisão do código.
    
3. O schema da section foi alterado para incluir uma opção de seleção entre **imagem** ou **vídeo**.
    
    Com base na opção escolhida pelo usuário, os campos correspondentes são exibidos ou ocultados no personalizador, evitando configurações inválidas ou conflitantes.
    
4. Foi criada uma validação em Liquid para identificar a opção selecionada no schema e renderizar corretamente o conteúdo escolhido (imagem ou vídeo).
5. Foi inserida a tag HTML `<video>` com os atributos de configuração adequados, incluindo:
    - `autoplay`
    - `loop`
    - `muted` (necessário para garantir o funcionamento do autoplay conforme as políticas dos navegadores modernos)
    - `playsinline`
6. O carregamento preguiçoso do vídeo foi implementado utilizando `IntersectionObserver`.
    
    O vídeo só tem seu `source` inserido quando entra na viewport, evitando o download desnecessário de mídia fora da tela.
    
    Para evitar a criação de múltiplos observers quando a section é reutilizada diversas vezes na página, o script foi encapsulado dentro da tag `{% javascript %}`.
    
    Essa abordagem permite que o Shopify consolide o script em um único arquivo, evitando execuções duplicadas e possíveis efeitos colaterais, como múltiplos observers associados ao mesmo elemento de vídeo.
    

---

### Imagem mobile na seção “Apresentação de slides” (Slideshow) – 42 minutos

### Requisitos

1. A imagem mobile deve ser editável via personalizador de tema.
2. A imagem desktop deve ser exibida em telas grandes (desktop e tablet).
3. A imagem mobile deve ser exibida exclusivamente em dispositivos móveis.

### Passos do desenvolvimento

1. A section Slideshow foi analisada no Theme Customizer para entender suas configurações existentes e garantir que nenhuma funcionalidade fosse quebrada durante a implementação.
2. Foi adicionada uma nova configuração no schema para permitir o upload de uma imagem específica para dispositivos móveis.
3. A exibição das imagens foi controlada utilizando media queries, garantindo que:
    - A imagem desktop seja exibida em telas maiores (desktop e tablet).
    - A imagem mobile seja exibida apenas em dispositivos móveis.

Essa abordagem garante flexibilidade para o cliente e melhora a experiência do usuário em diferentes tamanhos de tela, além de possibilitar otimizações de performance ao servir imagens adequadas para cada dispositivo.

---

## Recomendação de melhoria de desempenho web – 3 horas

### Contexto

Após a implementação das sections descritas acima, foram realizados testes de performance utilizando **Lighthouse** e **PageSpeed Insights**.

Os resultados apresentaram boa performance geral, o que já era esperado, considerando:

- A quantidade reduzida de sections na página
- A ausência de conteúdo pesado

---

### Análise de requisições

1. A análise foi aprofundada utilizando a aba **Network** do DevTools para identificar possíveis gargalos de carregamento.
2. Nenhuma requisição apresentou tempo excessivo de carregamento ou tamanho crítico de arquivo.
3. Uma possível melhoria identificada seria a **minificação dos arquivos CSS base**, principalmente aqueles que não sofrem alterações frequentes.
4. Outro ponto relevante é que alguns arquivos CSS poderiam ser movidos para dentro das próprias sections utilizando a tag `{% stylesheet %}`.
    
    Isso permitiria:
    
    - Melhor controle de escopo dos estilos
    - Redução de duplicação de CSS
    - Carregamento mais eficiente quando a section é reutilizada
    
    Como exemplo, a section Slideshow atualmente utiliza três arquivos CSS que são carregados repetidamente sempre que a section é adicionada à página.
    

---

### Lazy loading de sections utilizando Section Rendering API

Um ponto adicional que pode contribuir significativamente para a melhoria de performance é a implementação de **lazy loading por section**.

Durante a análise, foi identificado que a **Shopify Section Rendering API** pode ser uma aliada importante nesse cenário.

A ideia consiste em:

- Carregar inicialmente apenas as sections **above the fold**
- Utilizar `IntersectionObserver` para identificar quando uma section entra na viewport
- Buscar dinamicamente o conteúdo da section via API somente quando necessário

Essa abordagem é especialmente vantajosa para páginas com grande quantidade de sections, pois reduz o tempo de carregamento inicial e o consumo de recursos.

### Exemplo de implementação

**Theme.liquid**

```
{% if request.design_mode == false %}
<script>
  document.addEventListener('DOMContentLoaded', () => {
    const sections = document.querySelectorAll('.shopify-lazyload-section');

    const observer = new IntersectionObserver(async (entries, observer) => {
      for (const entry of entries) {
        if (entry.isIntersecting) {
          const section = entry.target;
          const sectionId = section.id.replace('shopify-section-template-', 'template-');

          try {
            const response = await fetch(
              `${window.Shopify.routes.root}?sections=${sectionId}&lazy_load=false`
            );
            const data = await response.json();

            if (data[sectionId]) {
              section.outerHTML = data[sectionId];
            }

            observer.unobserve(section);
          } catch (err) {
            console.error('Erro ao carregar section:', sectionId, err);
          }
        }
      }
    }, {
      threshold: 0.04,
    });

    sections.forEach(section => observer.observe(section));
  });
</script>
{% endif %}

```

---

**Section de exemplo**

```
{% capture lazy_load %}
  {% render 'get-page-url' %}
{% endcapture %}
{% assign lazy_load = lazy_load | strip %}

{% if lazy_load == 'false' or request.design_mode == true %}
  <style>
    #shopify-section-{{ section.id }} {
      --section-bg: {{ section.settings.bg_color }};
      --section-text: {{ section.settings.text_color }};
      --section-border: {{ section.settings.border_color }};
      --section-border-width: {{ section.settings.border_width }}px;
    }

    .test {
      background-color: var(--section-bg);
      color: var(--section-text);
      border: var(--section-border-width) solid var(--section-border);
    }
  </style>

  <div class="test">
    {{ section.id }}
    <h1>Test</h1>
  </div>
{% else %}
  <style>
    #shopify-section-{{ section.id }} {
      min-height: 400px;
    }
  </style>
{% endif %}

```

> Essa estratégia deve ser aplicada apenas a sections abaixo da dobra (below the fold), evitando impactos negativos em SEO, LCP e na experiência do usuário.
> 

---

### Considerações sobre preconnect

O uso de `preconnect` pode ser um aliado importante na melhoria de performance ao reduzir o tempo de estabelecimento de conexões externas.

No entanto, seu uso deve ser criterioso, pois conexões desnecessárias podem aumentar o **Total Blocking Time (TBT)** e impactar negativamente métricas como o **Largest Contentful Paint (LCP)**. O Tema não apresenta muito uso do preconect e então considero que adicionar mais preconects poderia impactar a performance.
