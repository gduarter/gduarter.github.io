---
title: "Dinâmica molecular de proteína e ligante em solvente "
excerpt: "Tutorial de dinâmica molecular para avaliar a estabilidade da pose de um ligante dentro do sítio ativo de uma proteína."
collection: portfolio
---

# Tutorial simulações proteína + ligante + solvente

Created: March 18, 2022 11:01 AM
Updated: December 16, 2022 9:53 PM

## **Introdução**

Este tutorial foi feito para auxiliar iniciantes na preparação de inputs para simulações de Dinâmica Molecular e Docking entre uma proteína e o seu ligante. 

Dinâmica molecular é o tipo de simulação é usada para avaliar a estabilidade do ligante dentro do sítio ativo e para explorar um pouco melhor as poses de ligação. Docking, por sua vez, é usado para verificar a capacidade que determinado ligante pode ter de se ligar em um receptor. Para mais detalhes sobre ambas as técnicas, veja material específico. 

---

## Obtenha a estrutura do complexo

Usando o UCSF Chimera ([https://www.cgl.ucsf.edu/chimera/](https://www.cgl.ucsf.edu/chimera/)), faça o download da estrutura do complexo em:

```
File > Fetch by ID 
```

Essa sequência abrirá uma janela em que é possível escolher o banco de dados de onde a estrutura será retirada. Normalmente o Protein Data Bank (PDB) basta, mas visto que ele está sendo descontinuado em prol de um banco de dados melhor, é bom se atentar às demais formas de conseguir a estrutura.

Use o Chimera para remover moléculas indesejadas (moléculas de água, surfactantes, íons etc) que não participem diretamente da interação do ligante (ou do cofator) com a proteína:

```
Select > Residue
```

Selecione tudo que não for o ligante, o cofator ou moléculas de água ou íons de importância para o sítio de ligação e remova usando

```
Actions > Atoms/Bonds > delete
```

Salve a estrutura como `${PDB}.clean.pdb`, onde `${PDB}` deve ser substituído pelo código PDB da molécula (e.g: `6me2.clean.pdb`, no exemplo [deste tutorial de docking](https://ringo.ams.stonybrook.edu/index.php/2022_DOCK_tutorial_1_with_PDBID_6ME2)).  Salve em pasta específica para conter as estruturas, como o diretório `001.initial_files` abaixo:

```bash
mkdir 001.initial_files     # Cria pasta
cd 001.initial_files        # Entra na pasta
```

Usando o Chimera, selecione o ligante:

```bash
Select > Residue 
```

E escolha a opção correspondente ao ligante no sítio ativo. Assim que a molécula estiver selecionada, vá em:

```bash
Select > Invert (all models)
```

o que causará a inversão da seleção: toda a proteína estará selecionada em detrimento do ligante. Usando o Chimera, apague a proteína da mesma forma que as moléculas inconvenientes foram apagadas:

```bash
Actions > Atoms/Bonds > delete
```

Observe que estruturas obtidas de Cristalografia de Raio-X não contém hidrogênios. O Chimera pode ser usado para isso:

```bash
Tools > Structure Editing > AddH
```

Uma janela abrirá com opções a respeito de estados de protonação e possíveis ligações de hidrogênio. A menos que saiba de algo muito específico a respeito do ligante, apenas confirme em `OK` e veja os hidrogênios adicionados. Observe que o Chimera automaticamente cria moléculas em estados de protonação fisiológicos ($R-NH_3^+$ ou $R-CO_2^-$, por exemplo). Para descobrir se a molécula é carregada, adicione cargas AM1-BCC usando o Chimera:

```bash
Tools > Structure Editing > Add Charge
```

Ao clicar OK na janela que aparecer, aparecerá uma outra janela para a especificação da carga. O programa é capaz de fazer uma boa suposição inicial, mas confira na literatura se faz sentido. Ao assentir pela segunda vez, o Chimera chamará o programa `Antechamber` para fazer a determinação das cargas atômicas. Quando terminado o cálculo, salve o arquivo em `001.initial_files` como `${PDB}.lig.mol2`:

```bash
File > Save Mol2
```

A despeito das diversas opções aparecerão na janela de salvar, escolha somente `Use untransformed coordinates` e `Use Sybyl-style hydrogen naming`. Faça o mesmo com o cofator, se houver, nomeando-o `${PDB}.cof.mol2`. 

Para salvar a proteína, feche a sessão e reabra `${PDB}.clean.pdb`. Selecione o ligante novamente e, desta vez, delete-o. Salve o receptor como `${PDB}.rec.pdb`. Adicione hidrogênios, cargas atômicas e salve, novamente, como `${PDB}.rec.mol2`. Caso precise fazer um experimento de docking, você usará o arquivo `mol2`.

---

## Preparar ligante e cofator usando o Antechamber

Tendo em posse o arquivo do ligante e do cofator, caso exista, prepare o sistema para dinâmica molecular:

```bash
antechamber -i ${PDB}.lig.mol2 -fi mol2 -o ${PDB}.lig.charged.mol2 -fo mol2 -at gaff2 -c bcc -rn LIG -nc ${charge} -pf y
```

```bash
antechamber -i ${PDB}.cof.mol2 -fi mol2 -o ${PDB}.cof.charged.mol2 -fo mol2 -at gaff2 -c bcc -rn COF -nc ${charge} -pf y
```

`-i` diz respeito ao *input* com a estrutura da molécula de interesse; `-fi` sinaliza o formato do *input*; `-o`, o nome do *output*; `-fo`, o formato do *output*. `-at` determina o campo de força a ser usado; `-c`, que tipo de cargas atômicas serão atribuídas a cada átomo. `-nc` corresponde a carga total e `-rn` é o código de 3 letras que o usuário dá a molécula. `${charge}` corresponde a carga da espécie, sempre expressa em múltiplos da carga de um elétron. `-pf` diz para o antechamber apagar arquivos intermediários do processo de cálculo de carga.

Após terminado, deve-se usar o programa `parmchk2` para verificar se está tudo em ordem com os parâmetros:

```bash
parmchk2 -i ${PDB}.lig.charged.mol2 -f mol2 -o ${PDB}.lig.charged.frcmod
```

No caso do cofator, se houver, deve-se usar o `antechamber` para criar um arquivo `prep` e um arquivo `pdb`, além do `frcmod`:

```bash
antechamber -i ${PDB}.cof.charged.mol2 -fi mol2 -o ${PDB}.cof.charged.prep -fo prepi -pf y
```

```bash
antechamber -i ${PDB}.cof.charged.mol2 -fi mol2 -o ${PDB}.cof.charged.pdb -fo pdb -pf y
```

```bash
parmchk2 -i ${PDB}.cof.charged.prep -f prepi -o ${PDB}.cof.charged.frcmod
```

## Preparar proteína e complexo usando o tleap

É importante não apagar os arquivos anteriores porque os resíduos de aminoácidos em estruturas oriundas do PDB estão numerados de acordo com regras preestabelecidas e os programas do pacote Ambertools, que contém o `Antechamber`, o `parmchk2` e o `tleap` renumeram os resíduos a partir do número 1.  

Deve-se tomar nota de alguns fatores importantes:

- Alguns resíduos podem estar modificados.
- Cisteínas podem fazer ligação dissulfídica.
- A estrutura da proteína pode estar incompleta.

Enquanto o `tleap` consegue adicionar terminais em partes da proteína que estão faltando, ele por vezes não funciona com resíduos modificados (que podem ser modificados para suas versões originais) e com cisteínas formando ligações dissulfídicas. Mudanças estruturais podem ser efetuadas no Chimera:

```bash
Tools > Structure Editing > Build Structure
```

ou manualmente mediante inserção e deleção. Cisteínas que formam ligação dissulfídica exigem modificação textual no arquivo `.pdb`. Os resíduos nomeados `CYS` que fazem ligação dissulfídica, devem ser modificados para `CYX`, o que permite a leitura pelo `tleap`.

Para criar um arquivo que será lido pelo `tleap`, crie o seguinte arquivo de texto:

```bash
set default PBradii mbondi2
source oldff/leaprc.ff14SB
loadamberparams frcmod.ions234lm_126_tip3p
loadamberparams frcmod.ions1lm_126_tip3p
loadamberparams frcmod.tip3p
REC = loadpdb ${PDB}.rec.pdb
charge REC
saveamberparm REC ${PDB}.pro.parm ${PDB}.pro.crd
savepdb REC ${PDB}.000.pdb
quit
```

Salve o arquivo como `pro.leap.in`.  Para executar o `tleap`, basta digitar no terminal:

```bash
tleap -f pro.leap.in
```

### Significado das linhas do arquivo de entrada do `tleap`

O arquivo `pro.leap.in` contém informações importantes para a construção do sistema:

- `set default PBradii mbondi2` define raios de Poisson-Boltzmann no caso de que se queira calcular energias livres por meio de técnicas menos sofisticadas como MM-PBSA ou MM-GBSA.
- `source oldff/leaprc.ff14SB` lê o arquivo de parâmetros do campo de força Amber ff14SB, um bom campo de forças para proteínas e ácidos nucleicos.
- As linhas `loadamberparams frcmod.ions234lm_126_tip3p`, `loadamberparams frcmod.ions1lm_126_tip3p` e `loadamberparams frcmod.tip3p` lêem arquivos contendo parâmetros para íons e água (modelo rígido `tip3p`).
- `REC = loadpdb ${PDB}.rec.pdb` lê o arquivo `.pdb` .
- `charge REC` confere a cada átomo uma carga de acordo com o campo de força escolhido.
- As linhas `saveamberparm REC ${PDB}.pro.prmtop ${PDB}.pro.inpcrd` e `savepdb REC ${PDB}.000.pdb` criam arquivos `.prmtop` e `.inpcrd` para simulações no `Sander` ou `PME-MD` (programas de dinâmica molecular do pacote Amber) e um arquivo PDB que pode ser usado para manter controle da numeração biológica e sua correspondência com a numeração criada pelo `tleap`.
- `quit` fecha o programa.

### Numeração biológica

Há um programa do pacote Amber que pode adicionar ao `.prmtop` a numeração biológica, `parmed`. Para fazer os números biológicos serem incluídos crie um arquivo `parmed.in` com a seguinte linha:

```bash
addPDB ${PDB}.rec.pdb
```

onde `${PDB}.rec.pdb` é o pdb com a numeração biológica correta. Para colocar a informação biológica nos arquivos gerados pelo `tleap` use:

```bash
parmed -i parmed.in -p ${PDB}.pro.prmtop -c ${PDB}.pro.inpcrd
```

Isso nem sempre funcionará, então fique atento aos principais resíduos do sítio de ligação para fazer a sua análise.

### Criando o complexo proteína+ligante+cofator

Para criar o sistema para simulação, usamos o `tleap` para criar os arquivos de simulação do pacote Amber. Para isso criamos o input `${PDB}.leap.in`:

```bash
set default PBradii mbondi3
source oldff/leaprc.ff14SB
source leaprc.gaff2
loadamberparams frcmod.ions234lm_126_tip3p
loadamberparams frcmod.ions1lm_126_tip3p
loadamberparams frcmod.tip3p
### Adicionar proteína (PRO)
PRO = loadpdb ${path_to_place}/${PDB}.pro.noH.pdb
### Adicionar ligante (LIG)
loadamberparams ${PDB}.lig.charged.frcmod
LIG = loadmol2 ${PDB}.lig.charged.mol2
### Adicionar cofator (COF), se ele existir. 
### Caso não exista cofator, apague as linhas com `cof`
loadamberparams ${path_to_place}/${PDB}.cof.charged.frcmod
loadamberprep ${path_to_place}/${PDB}.cof.charged.prep
COF = loadpdb ${path_to_place}/${PDB}.cof.charged.pdb
REC = combine { PRO COF }
saveamberparm COF ${PDB}.cof.prmtop ${PDB}.cof.ori.inpcrd
### Criar complexo
COM = combine { REC LIG }
### A linha abaixo deve ser usada caso não exista cofator
#COM = combine { PRO LIG }
saveamberparm LIG ${PDB}.lig.prmtop ${PDB}.lig.ori.inpcrd
saveamberparm PRO ${PDB}.pro.prmtop ${PDB}.pro.ori.inpcrd
saveamberparm REC ${PDB}.rec.prmtop ${PDB}.rec.ori.inpcrd
saveamberparm COM ${PDB}.com.prmtop ${PDB}.com.ori.inpcrd
### Cria complexo para solvatação (passo redundante, mas OK)
solvcomplex= combine { REC LIG }
### solvatação do sistema
solvateoct solvcomplex TIP3PBOX 12.0
### neutralização do sistema (adiciona Na ou Cl dependendo da carga)
addions solvcomplex Cl- 0
addions solvcomplex Na+ 0
### escrever arquivo PDB com estrutura solvatada
savepdb solvcomplex ${PDB}.com.solv.pdb
### Verificar o sistema
charge solvcomplex
check solvcomplex
### Escrever arquivos de topologia e coordenadas
saveamberparm solvcomplex ${PDB}.com.solv.prmtop ${PDB}.com.solv.inpcrd
quit
```

e executamos o comando `tleap -f ${PDB}.leap.in` para criar dois arquivos, `${PDB}.com.solv.prmtop` e `${PDB}.com.solv.inpcrd`. Esses são os arquivos de entrada para simulações de dinâmica molecular nos programas `sander` e `pmemd`, do pacote Amber. Não é o nosso caso. Para criarmos as coordenadas e topologias legíveis pelo Gromacs, usamos o script `amber_to_gmx.py`:

```bash
python amber_to_gmx.py -p ${PDB}.com.solv.prmtop -c ${PDB}.com.solv.inpcrd
```

que criará os arquivos `${PDB}.top` e `${PDB}.gro` que podem ser usados pelo GROMACS.

---

## Detalhes antes da simulação de dinâmica molecular

A geração de dados de Dinâmica Molecular não é imediata: como a estrutura é obtida a partir do resultado de um experimento de cristalografia de raios-X, precisamos relaxar a estrutura para que ela assuma uma configuração condizente com as condições desejadas.

São pelo menos três os estágios para a produção de uma boa trajetória de dinâmica
molecular:

### Minimização

A energia em função das coordenadas atômicas no sistema é
minimizada de forma a reduzir efeitos energéticos indesejáveis de impedimentos
estéreos. Sem minimização, simulações de dinâmica molecular podem dar errado
("explodir"/"*blow up*") devido a forças elevadas que poderiam ter sido atenuadas
com a minimização.

Para fazer a minimização, digite no terminal:

```
gmx grompp -f minimization.mdp -c solute_in_solvent.gro -p solute_in_solvent.top -o minimization.tpr
gmx mdrun -deffnm minimization

```

O primeiro comando organiza a simulação, o segundo faz a dinâmica molecular ou
a minimização rodar no seu computador. A *flag* `-f` indica o arquivo `.mdp`,
`-c` indica o arquivo `.gro` e `-p` indica o arquivo `.top`. O *output* de
interesse para nós é `minimization.gro` (nomeado pela *flag* `-deffnm`), que
contém a estrutura minimizada do sistema de interesse. `-o` é o *output* com o
sistema pronto para o programa `mdrun`.

### Equilibração

Após a minimização a energia se encontra em um mínimo e o sistema
não necessariamente se encontra em uma configuração representativa. Além disso,
em dinâmica molecular, estamos interessados em propriedades de *ensemble* e, mesmo
se a configuração inicial fosse quimicamente relevante, ela não teria o devido
valor estatístico.

São três os estágios de equilibração:

- **Equilibração de temperatura:** `equil_nvt.mdp` define o estágio de equilibração
da temperatura. Essa etapa é necessária porque a estrutura minimizada corresponderia
ao sistema na temperatura de $0 \text{kelvin}$. Além disso, é no início desta etapa
que velocidades são atribuídas a cada átomo, o que é imprescindível para a simulação.
Para rodar esse estágio, lembre-se que a estrutura de partida está definida no
*output* da minimização:

```
gmx grompp -f equil_nvt.mdp -c minimization.gro -p solute_in_solvent.top -o equil_nvt.tpr
gmx mdrun -deffnm equil_nvt
```

- **Equilibração de pressão 1:** `equil_npt.mdp` define o primeiro estágio de
equilibração da pressão. São necessários dois estágios porque o barostato de Berendsen
usado neste estágio, ajuda a caixa de simulação ficar com um volume adequado, mas
não produz um ensemble isotérmico-isobárico adequado. Para isso necessitamos de um
segundo estágio de equilibração da pressão. Para isso, usamos o arquivo `.gro`
produzido pela etapa de equilíbrio da temperatura, pois cada átomo além de conter
suas coordenadas atômicas, também tem as componentes de sua velocidade em uma
temperatura adequada:

```
gmx grompp -f equil_npt.mdp -c equil_nvt.gro -p solute_in_solvent.top -o equil_npt.tpr
gmx mdrun -deffnm equil_npt
```

- **Equilibração de pressão 2:** `equil_npt2.mdp` define o segundo estágio de
equilibração da pressão. Neste estágio, usamos o barostato de Parrinello-Rahman,
que produz um ensemble isotérmico-isobárico adequado. A diferença entre o estágio
anterior e este é que o comprimento dos lados da caixa podem variar de forma
independente, enquanto isso não era o caso do estágio anterior. Para obter os
resultados:

```
gmx grompp -f equil_npt2.mdp -c equil_npt.gro -p solute_in_solvent.top -o equil_npt2.tpr
gmx mdrun -deffnm equil_npt2
```

### Produção

O estágio mais importante de uma simulação de dinâmica molecular é a amostragem
das configurações de equilíbrio do sistema. Isso é feito na etapa de produção.`prod.mdp` define este estágio. A produção é significantemente maior que os
demais passos; a amostragem de configurações de moléculas pequenas em água
costuma exigir 2500000 passos, implicando em um tempo computacional de algumas
horas.

A produção resulta na criação de uma trajetória contendo configurações no *ensemble*
isotérmico-isobárico (NPT) que podem ser usadas para calcular propriedades de
equilíbrio do sistema. A simulação é rodada com os seguintes comandos:

```
gmx grompp -f prod.mdp -c equil_npt2.gro -p solute_in_solvent.top -o prod.tpr
gmx mdrun -deffnm prod
```

Todos os dados da trajetória estão armazenados em arquivos `.trr` e
`.xtc` que podem ser analisados com outros programas incluídos no GROMACS. Energias
e outras grandezas físicas podem ser retiradas de arquivos `.edr`.

A dinâmica molecular no GROMACS necessita de arquivos `.mdp` contendo as instruções para a simulação. 

### Sistemas contendo proteínas

No caso de simulações contendo proteínas e ligantes, alguns cuidados a mais devem ser tomados.

O primeiro estágio da equilibração, por exemplo, exige que a proteína e o ligante tenham restrições de movimento para que as moléculas de água que envelopam o sistema se assentem. O GROMACS exige que essas restrições sejam incluídas nos arquivos `.top` após a definição dos últimos parâmetros de campo de força de cada molécula. A diretiva:

```bash
#ifdef POSRES
#include "${nome_do_arquivo_de_restrição}.itp"
#endif
```

normalmente é incluída no final da definição de cada espécie, antes da nova seção `moleculetype` das topologias. Para criar os arquivos `.itp`, executamos o seguinte programa do pacote GROMACS:

```bash
gmx genrestr -f ${PDB}.gro -o posres.itp 
```

É costume selecionar a opção `Protein-H`, isto é, todos os átomos da proteína que não sejam hidrogênio. Para fazer o mesmo com o ligante é necessário primeiramente criar um arquivo de índice que indica somente os átomos que não são hidrogênio:

```bash
gmx make_ndx -f ${PDB}.gro -o index_lig.ndx
```

No programa `gmx make_ndx` você escolhe a opção do ligante:

```bash
> ${número do ligante} & ! a H*
> q
```

onde `${número do ligante}` é uma das opções impressas na tela. Esse comando selecionará somente os átomos que não sejam hidrogênio do ligante e os distinguirá no arquivo `index_lig.ndx`. Você pode criar as restrições para esses átomos usando o `gmx genrestr`:

```bash
gmx genrestr -f ${PDB}.gro -n index_lig.ndx -o posre_lig.itp
```

selecionando a opção que você gerou (algo como `LIG_&_!H*`. Isso gerará o arquivo desejado. Caso haja cofatores, é possível considerá-los parte da proteína, basta fazer as modificações textuais adequadas nos arquivos `.ndx` gerados.

**Importante!**

Os arquivos `.itp` precisam ser numerados a partir de 1 e não do número do `.gro` do sistema. Isso deve ser modificado para que a simulação rode. Você deve gerar um arquivo `.gro` (use `gmx trjconv` com o `.gro` e um `.tpr`) contendo somente a molécula de interesse e usar `gmx genrestr` para gerar o arquivo `.itp` da forma correta. Se for uma subestrutura da proteína, às vezes pode ser necessário renumerar usando um script de python específico.

```markdown
Apagar isso depois de escrever o script:
- Subtrair (primeiro_valor - 1) de todos os números da primeira coluna
do arquivo `.itp`, onde primeiro_valor é o número do primeiro átomo.
```

---

## Rodando uma dinâmica molecular

Tendo os arquivos `.gro` e `.top` você pode rodar as simulações de dinâmica molecular usando os comandos mostrados na sessão anterior. Todos os detalhes da dinâmica estão contidos nos arquivos `.mdp`. Na **minimização**, por exemplo:

```bash
; Parametros sobre o que fazer, quando parar e o que salvar
integrator      = steep         ; Algoritmo de minimizacao Steepest Descent)
emtol           = 1000.0        ; Parar minimizacao quando F < 10.0 kJ/mol
emstep          = 0.01          ; Tamanho do passo na minimização
nsteps          = 50000         ; Número máximo de passos de minimização

; Parametros descrevendo como achar atomos vizinhos e como calcular suas interacoes
nstlist         = 1             ; Frequencia de atualizacao da lista de vizinhos
cutoff-scheme   = Verlet
ns_type         = grid          ; Metodo para determinar a lista de vizinhos (simple, grid)
rlist           = 1.2           ; Cut-off para fazera lista de vizinhos (forcas de curto alcance)
coulombtype     = PME           ; Tratamento de interacoes eletrostaticas de longo alcance
rcoulomb        = 1.2           ; Cut-off de longo alcance eletrostatico
vdwtype         = cutoff
vdw-modifier    = force-switch
rvdw-switch     = 1.0
rvdw            = 1.2           ; Cut-off de longo alcance de van der Waals
pbc             = xyz           ; Periodic Boundary Conditions
DispCorr        = AllEnerPres  ; correcao de dispersão
```

Na **equilibração NVT**:

```bash
;protein-ligand complex NVT equilibration
define                  = -DPOSRES -DPOSRES_LIG ; position restrain the protein and ligand
; Run parameters
integrator              = sd        ; integrador de Langevin
nsteps                  = 50000     ; 2 * 50000 = 100 ps
dt                      = 0.002     ; 2 fs
; Output control
nstenergy               = 1000   ; salvar energias a cada 2.0 ps
nstlog                  = 1000   ; atualizar log a cada 2.0 ps
nstxout-compressed      = 1000   ; salvar coordenadas a cada 2.0 ps
; Bond parameters
continuation            = no        ; Como e primeira dinamica, nao e continuacao
constraint_algorithm    = lincs     ; restricoes holonomicas
constraints             = hbonds    ; ligacoes com o H devem ser restringidas
lincs_iter              = 1         ; precisao do LINCS
lincs_order             = 4         ; parametro do LINCS
; Neighbor searching and vdW
cutoff-scheme           = Verlet
ns_type                 = grid      ; Metodo para determinar a lista de vizinhos (simple, grid)
nstlist                 = 20        ; 
rlist                   = 1.2
vdwtype                 = cutoff
vdw-modifier            = force-switch
rvdw-switch             = 1.0
rvdw                    = 1.2       ; Cut-off de curto alcance de vdW
; Electrostatics
coulombtype             = PME       ; Particle Mesh Ewald para eletrostatica de longo alcance
rcoulomb                = 1.2       ; Cut-off eletrostatico de curto alcance (in nm)
pme_order               = 4         ; interpolacao cubica
fourierspacing          = 0.16      ; espacamento do grid para FFT (Fast Fourier Transform)
; Temperature coupling
tcoupl                  = no        ; Langevin dynamics demanda acoplamento com termostato
tc-grps                 = System    ; nao necessario
tau_t                   = 2.0       ; constante de tempo em ps
ref_t                   = 298.15    ; temperatura de referencia em K
; Pressure coupling
pcoupl                  = no        ; sem barostato em NVT
; Periodic boundary conditions
pbc                     = xyz       ; PBC em tres dimensoes
DispCorr                = AllEnerPres
; Velocity generation
gen_vel                 = yes       ; velocidades iniciais oriundas da distribuicao de Maxwell-Boltzmann
gen_temp                = 298.15    ; Temperatura para a distribuicao de Maxwell
gen_seed                = -1        ; semente aleatoria gerada pelo programa
```

Na **equilibração NPT**:

```bash
define                  = -DPOSRES -DPOSRES_LIG
; Run parameters
integrator              = sd        ; Integrador de Langevin
nsteps                  = 50000     ; 2 * 50000 = 100 ps
dt                      = 0.002     ; 2 fs
; Output control
nstenergy               = 1000
nstlog                  = 1000
nstxout-compressed      = 1000
; Bond parameters
continuation            = yes       ; continuação do NVT
constraint_algorithm    = lincs
constraints             = hbonds
lincs_iter              = 1
lincs_order             = 4
; Neighbor searching and vdW
cutoff-scheme           = Verlet
ns_type                 = grid
nstlist                 = 20
rlist                   = 1.2
vdwtype                 = cutoff
vdw-modifier            = force-switch
rvdw-switch             = 1.0
rvdw                    = 1.2
; Electrostatics
coulombtype             = PME
rcoulomb                = 1.2
pme_order               = 4
fourierspacing          = 0.16
; Temperature coupling
tcoupl                  = no
tc-grps                 = System
tau_t                   = 2.0
ref_t                   = 298.15
; Pressure coupling
pcoupl                  = Berendsen    ; algoritmo do barostato para NPT
pcoupltype              = isotropic    ; caixa de simulacao varia uniformemente
tau_p                   = 2.0          ; constante de tempo
ref_p                   = 1.0          ; pressao de referencia em bar
compressibility         = 4.5e-5       ; compressibilidade isotermica da agua, bar^-1
refcoord_scaling        = com
; Periodic boundary conditions
pbc                     = xyz       ; 3-D PBC
DispCorr                = AllEnerPres
; Velocity generation
gen_vel                 = no        ; usar velocidades geradas da etapa NVT
ld_seed                 = -1
```

Na **equilibração NPT2**:

```bash
define                  = -DPOSRES -DPOSRES_LIG
; Run parameters
integrator              = sd        ; Integrador de Langevin
nsteps                  = 50000     ; 2 * 50000 = 100 ps
dt                      = 0.002     ; 2 fs
; Output control
nstenergy               = 1000
nstlog                  = 1000
nstxout-compressed      = 1000
; Bond parameters
continuation            = yes       ; continuação do NPT
constraint_algorithm    = lincs
constraints             = hbonds
lincs_iter              = 1
lincs_order             = 4
; Neighbor searching and vdW
cutoff-scheme           = Verlet
ns_type                 = grid
nstlist                 = 20
rlist                   = 1.2
vdwtype                 = cutoff
vdw-modifier            = force-switch
rvdw-switch             = 1.0
rvdw                    = 1.2
; Electrostatics
coulombtype             = PME
rcoulomb                = 1.2
pme_order               = 4
fourierspacing          = 0.16
; Temperature coupling
tcoupl                  = no
tc-grps                 = System
tau_t                   = 2.0
ref_t                   = 298.15
; Pressure coupling
pcoupl                  = Parrinello-Rahman    ; algoritmo do barostato para NPT
pcoupltype              = isotropic            ; caixa de simulacao varia uniformemente
tau_p                   = 5.0                  ; constante de tempo
ref_p                   = 1.0                  ; pressao de referencia em bar
compressibility         = 4.5e-5               ; compressibilidade isotermica da agua, bar^-1
refcoord_scaling        = com                  ; permite o uso de restricoes de posicao com o barostato
; Periodic boundary conditions
pbc                     = xyz       ; 3-D PBC
DispCorr                = AllEnerPres
; Velocity generation
gen_vel                 = no        ; usar velocidades geradas da etapa NPT
```

Às vezes compensa rodar uma etapa de **equilibração NPT3** em que as cadeias laterais dos resíduos possam se mover e acomodar melhor ao redor do ligante. Um arquivo `backbone.itp` deve ser criado antes disso:

```bash
define                  = -DBACKBONE -DPOSRES_LIG
; Run parameters
integrator              = sd        ; Integrador de Langevin
nsteps                  = 50000     ; 2 * 50000 = 100 ps
nstenergy               = 1000
nstlog                  = 1000
nstxout-compressed      = 1000
; Bond parameters
continuation            = yes       ; continuação do NPT
constraint_algorithm    = lincs
constraints             = hbonds
lincs_iter              = 1
lincs_order             = 4
; Neighbor searching and vdW
cutoff-scheme           = Verlet
ns_type                 = grid
nstlist                 = 20
rlist                   = 1.2
vdwtype                 = cutoff
vdw-modifier            = force-switch
rvdw-switch             = 1.0
rvdw                    = 1.2
; Electrostatics
coulombtype             = PME
rcoulomb                = 1.2
pme_order               = 4
fourierspacing          = 0.16
; Temperature coupling
tcoupl                  = no
tc-grps                 = System
tau_t                   = 2.0
ref_t                   = 298.15
; Pressure coupling
pcoupl                  = Parrinello-Rahman    ; algoritmo do barostato para NPT
pcoupltype              = isotropic            ; caixa de simulacao varia uniformemente
tau_p                   = 5.0                  ; constante de tempo
ref_p                   = 1.0                  ; pressao de referencia em bar
compressibility         = 4.5e-5               ; compressibilidade isotermica da agua, bar^-1
refcoord_scaling        = com                  ; permite o uso de restricoes de posicao com o barostato
; Periodic boundary conditions
pbc                     = xyz       ; 3-D PBC
DispCorr                = AllEnerPres
; Velocity generation
gen_vel                 = no        ; usar velocidades geradas da etapa NPT
```


Por último, temos a etapa de **produção**, que tem os mesmos parâmetros da etapa de equilibração NPT2, mas nenhuma restrição aplicada no sistema.

```bash
; Run parameters
integrator              = sd        ; Integrador de Langevin
nsteps                  = 5000000   ; 10 ns
dt                      = 0.002     ; 2 fs
; Output control
nstenergy               = 1000
nstlog                  = 1000
nstxout-compressed      = 1000
; Bond parameters
continuation            = yes       ; continuação do NPT
constraint_algorithm    = lincs
constraints             = hbonds
lincs_iter              = 1
lincs_order             = 4
; Neighbor searching and vdW
cutoff-scheme           = Verlet
ns_type                 = grid
nstlist                 = 20
rlist                   = 1.2
vdwtype                 = cutoff
vdw-modifier            = force-switch
rvdw-switch             = 1.0
rvdw                    = 1.2
; Electrostatics
coulombtype             = PME
rcoulomb                = 1.2
pme_order               = 4
fourierspacing          = 0.16
; Temperature coupling
tcoupl                  = no
tc-grps                 = System
tau_t                   = 2.0
ref_t                   = 298.15
; Pressure coupling
pcoupl                  = Parrinello-Rahman    ; algoritmo do barostato para NPT
pcoupltype              = isotropic            ; caixa de simulacao varia uniformemente
tau_p                   = 5.0                  ; constante de tempo
ref_p                   = 1.0                  ; pressao de referencia em bar
compressibility         = 4.5e-5               ; compressibilidade isotermica da agua, bar^-1
refcoord_scaling        = com                  ; permite o uso de restricoes de posicao com o barostato
; Periodic boundary conditions
pbc                     = xyz                  ; 3-D PBC
DispCorr                = AllEnerPres
; Velocity generation
gen_vel                 = no                   ; usar velocidades geradas da etapa NPT
```
