# sql

SELECT 
    f.fornecedor AS FORNECEDOR,
    a.codprod AS CÓDIGO,       
    b.descricao AS DESCRIÇÃO,  
    LISTAGG(a.qtini || ' a ' || a.qtfim, ' | ') WITHIN GROUP (ORDER BY a.qtini) AS ESCALA,  
    LISTAGG(
        TO_CHAR(
            CASE 
                WHEN SUBSTR('S', 1, 1) = 'S' THEN 
                    CASE 
                        WHEN a.ALTERAPTABELA = 'N' THEN 
                            TO_NUMBER(NVL(c.PVENDA1, 0)) - (TO_NUMBER(NVL(c.PVENDA1, 0)) * NVL(a.PERCDESC, 0) / 100)  -- Preço mínimo sem alteração de tabela
                        ELSE 
                            (TO_NUMBER(NVL(c.PVENDA1, 0)) - ABS(TO_NUMBER(NVL(c.PVENDA1, 0)) * a.PERCDESC / 100)) 
                            - ((TO_NUMBER(c.PVENDA1) - ABS(TO_NUMBER(NVL(c.PVENDA1, 0)) * a.PERCDESC / 100)) 
                               * NVL(c.PERDESCMAXTAB, 0) / 100)  -- Preço mínimo considerando a máxima tabela
                    END 
                ELSE NULL 
            END, '999.99'), 
        ' | ') WITHIN GROUP (ORDER BY a.qtini) AS PREÇO_NET,
    
    -- Verifica se o codprod pertence a uma subcategoria
    CASE
        WHEN EXISTS (SELECT 1 FROM PCPRODUT p WHERE p.codprod = a.codprod AND p.codsubcategoria IS NOT NULL) 
        THEN 'Subcategoria Encontrada'
        ELSE 'Sem Subcategoria'
    END AS SUBCATEGORIA,
    
    -- Verifica se o produto está dentro do código do grupo da restrição
    CASE
        WHEN EXISTS (
            SELECT 1 
            FROM PCGRUPOSCAMPANHAI g 
            WHERE g.CODITEM = a.codprod 
            AND g.CODGRUPO = '164'
        ) THEN 'Grupo Restrito Encontrado'
        ELSE 'Sem Grupo Restrito'
    END AS CODGRUPO_RESTRITO

FROM 
    PCDESCONTO a
LEFT JOIN
    PCPRODUT b ON a.codprod = b.codprod
LEFT JOIN 
    PCFORNEC f ON b.codfornec = f.codfornec
LEFT JOIN 
    PCTABPR c ON b.codprod = c.codprod
WHERE 
    a.dtinicio >= TO_DATE('01/01/2024', 'DD/MM/YYYY')  
    AND c.numregiao = '1'
    AND b.obs2 <> 'FL'  
    AND LENGTH(a.codprod) = 4  
    AND a.codcli IS NULL 
    AND a.codrede IS NULL 
    AND a.qtini IS NOT NULL 
GROUP BY 
    f.fornecedor, a.codprod, b.descricao
ORDER BY 
    f.fornecedor, a.codprod;