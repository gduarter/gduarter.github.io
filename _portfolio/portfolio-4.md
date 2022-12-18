---
title: "Preparação para docking com o DOCK 6"
excerpt: "Tutorial de preparação para docking de ligante em proteína "
collection: portfolio
---

## Preparação do sistema para Docking usando DOCK6

### Organização
Crie alguns diretórios:

- `001.initial_files`
- `002.lig_prep`
- `003.rec_prep`
- `004.complex_prep`
- `005.gmx_prep`
- `006.dockprep`
- `007.spheres`
- `008.grid`

Salve o arquivo `.pdb` e todas as informações iniciais importantes na pasta `001.initial_files`. A pasta `002.lig_prep` será usada para armazenar todas as estruturas com átomos de hidrogênio e cargas parciais para uma equilibração em dinâmica molecular e o experimento de docking.

### Preparação de ligante e cofator, se existir
Use o UCSF Chimera para isolar a estrutura do receptor, do ligante e do cofator como

- `${PDB}.pro.pdb`, para a proteína
- `${PDB}.lig.pdb`, para o ligante
- `${PDB}.cof.pdb`, para o cofator, caso exista.

Salve esses arquivos em `001.initial_files`

Ainda usando o Chimera, adicione átomos de hidrogênio em todas as estruturas e as renomeie de acordo:

- `${PDB}.lig.withH.pdb`, para o ligante
- `${PDB}.cof.withH.pdb`, para o cofator, caso exista.

*Observação:* Caso não haja cofator, renomeie o arquivo do receptor proteico para `${PDB}.rec.pdb`

Crie arquivos `.mol2` para o ligante usando o programa Antechamber do pacote AmberTools:

```
antechamber -fi pdb -fo mol2 -c bcc -at sybyl -i ${PDB}.lig.withH.pdb -o ${PDB}.lig.charged.mol2 -rn LIG -nc ${carga}.
```

Observe que `${carga}` deve ser substituída pela carga do ligante e o mesmo procedimento deve ser feito para o cofator, caso exista. O script `fix_charges.py` pode ser usado para consertar pequenas eventuais falhas de ajuste do programa `sqm` associado ao Antechamber.

```bash
python ${path_to_script}/fix_charges.py -m ${PDB}.lig.charged.mol2
```

*Importante!* A ferramenta `dockprep` do UCSF Chimera pode determinar a carga total do ligante e preparar os arquivos da simulação, mas não é recomendado para trabalhos com vistas a publicações. Se for usá-lo, use para dados preliminares.

### Preparação do receptor

O preparo do receptor exige uma etapa de minimização, equilibração NVT e minimização com a presença do ligante (e do cofator) para que as cadeias laterais se acomodem apropriadamente. Quando o PDB em questão tem resolução < 2 angström, somente a minimização basta. Recomenda-se que estruturas obtidas de cryo-EM passem pelas três etapas de preparo.

Entre no diretório `003.rec_prep` e crie um arquivo para ser lido pelo `tleap`:
```bash
cat <<EOF > ${PDB}.rec.leap.in
set default PBradii mbondi2
source oldff/leaprc.ff14SB
source leaprc.DNA.bsc1
loadamberparams frcmod.ions234lm_126_tip3p
loadamberparams frcmod.ions1lm_126_tip3p
loadamberparams frcmod.tip3p
REC = loadpdb ${path_to_file}/${PDB}.rec.noH.pdb
saveamberparm REC ${PDB}.rec.prmtop ${1PDB}.rec.inpcrd
charge REC
quit
EOF
```
Ao executar o `tleap`, serão gerados dois arquivos, `${PDB}.rec.prmtop` and `${PDB}.rec.inpcrd`:
```
tleap -f ${PDB}.rec.leap.in
```
Leia atentamente as mensagens de erro, pois elas indicam modificações que você precisará fazer no arquivo `.pdb`.

*Atenção!* Se o `tleap` reclamar algo como:
```
/Users/guilherme/miniconda3/envs/py37/bin/teLeap: Error!
Could not find bond parameter for: SH - SH
```
ele está tendo problemas para diferenciar cisteínas que fazem ligações dissulfídicas de cisteínas que não fazem tais ligações. Você deve renomear as cisteínas que fazem ligações S-S de `CYS` para `CYX`. Para descobrir quais cisteínas formam tal ligação, use o UCSF Chimera, selecione os resíduos CYS e mostre as cadeias laterais.

Outra mensagem que pode aparecer e deve ser lidada é a presença de avisos ("Warnings") do tipo:
```
/Users/guilherme/miniconda3/envs/py37/bin/teLeap: Warning!
There is a bond of 11.508708 angstroms between:
```
Isso é facilmente resolvido com a adição de uma linha `TER` entre os resíduos em questão no arquivo `.pdb`. Em alguns casos é necessário adicionar grupos neutros (`ACE` e `NME`) nos terminais, mas deve-se evitá-los a menos que seja necessário (quando estamos interessados em estudar um fragmento específico, por exemplo).

`tleap` informa a carga total da proteína em um aviso e no final da sua execução:
```
/Users/guilherme/miniconda3/envs/py37/bin/teLeap: Warning!
The unperturbed charge of the unit (1.000000) is not zero.
```
e
```
Total unperturbed charge:   1.000000
Total perturbed charge:     1.000000
    Quit
```
*Atenção!* É importante guardar o `.pdb` original porque o `tleap` renumera todos os resíduos!
Esta etapa é necessária para conhecermos as características principais da proteína e consertar quaisquer problemas diretamente no `.pdb`.

### Preparação do complexo

Entre em `004.complex_prep` na pasta do projeto.
De forma semelhante à preparação do receptor, para preparar o complexo, precisamos criar um input para o `tleap`:
```bash
cat <<EOF > com.leap.in
set default PBradii mbondi2
source oldff/leaprc.ff14SB
source leaprc.gaff2
loadamberparams frcmod.ions234lm_126_tip3p
loadamberparams frcmod.ions1lm_126_tip3p
loadamberparams frcmod.tip3p
REC = loadpdb ../001.initial_files/${PDB}.rec.noH.pdb
loadamberparams ${PDB}.lig.antechamber.frcmod
LIG = loadmol2 ${PDB}.lig.antechamber.mol2
COM = combine { REC LIG }
saveamberparm LIG ${PDB}.lig.prmtop ${PDB}.lig.ori.inpcrd
saveamberparm REC ${PDB}.rec.prmtop ${PDB}.rec.ori.inpcrd
saveamberparm COM ${PDB}.com.prmtop ${PDB}.com.ori.inpcrd
solvcomplex= combine { REC LIG }
solvateoct solvcomplex TIP3PBOX 12.0
addions solvcomplex Cl- 0
addions solvcomplex Na+ 0
savepdb solvcomplex ${PDB}.com.solv.pdb
charge solvcomplex
check solvcomplex
saveamberparm solvcomplex ${PDB}.com.solv.prmtop ${PDB}.com.solv.inpcrd
quit
EOF
```
Observe, entretanto, que devemos fazer algumas modificações com relação ao ligante. Precisamos de um arquivo `.frcmod` e uma versão do `.mol2` que não use tipos atômicos SYBYL, mas tipos compatíveis com o Amber. Para isso:
```
antechamber -i ../002.lig_prep/${PDB}.lig.charged.mol2 -fi mol2 -o ${PDB}.lig.antechamber.mol2 -fo mol2 -dr n
parmchk2 -i ${PDB}.lig.antechamber.mol2 -f mol2 -o ${PDB}.lig.antechamber.frcmod
```

Se houver cofator:
```bash
cat <<EOF > com.leap.in
set default PBradii mbondi2
source oldff/leaprc.ff14SB
source leaprc.gaff2
loadamberparams frcmod.ions234lm_126_tip3p
loadamberparams frcmod.ions1lm_126_tip3p
loadamberparams frcmod.tip3p
PRO = loadpdb ../001.initial_files/${PDB}.pro.noH.pdb
loadamberparams ${PDB}.lig.antechamber.frcmod
LIG = loadmol2 ${PDB}.lig.antechamber.mol2
loadamberparams ${PDB}.cof.antechamber.frcmod
loadamberprep ${PDB}.cof.antechamber.prep
COF = loadpdb ${PDB}.cof.antechamber.pdb
REC = combine { PRO COF }
saveamberparm COF ${PDB}.cof.prmtop ${PDB}.cof.ori.inpcrd
COM = combine { REC LIG }
saveamberparm LIG ${PDB}.lig.prmtop ${PDB}.lig.ori.inpcrd
saveamberparm PRO ${PDB}.pro.prmtop ${PDB}.pro.ori.inpcrd
saveamberparm REC ${PDB}.rec.prmtop ${PDB}.rec.ori.inpcrd
saveamberparm COM ${PDB}.com.prmtop ${PDB}.com.ori.inpcrd
solvcomplex= combine { REC LIG }
solvateoct solvcomplex TIP3PBOX 12.0
addions solvcomplex Cl- 0
addions solvcomplex Na+ 0
savepdb solvcomplex ${PDB}.com.solv.pdb
charge solvcomplex
check solvcomplex
saveamberparm solvcomplex ${PDB}.com.solv.prmtop ${PDB}.com.solv.inpcrd
quit
EOF
```
e também:
```
antechamber -i ../002.lig_prep/${PDB}.cof.charged.mol2 -fi mol2 -o ${PDB}.cof.antechamber.prep -fo prepi
antechamber -i ../002.lig_prep/${PDB}.cof.charged.mol2 -fi mol2 -o ${PDB}.cof.antechamber.pdb -fo pdb
parmchk2 -i ${PDB}.cof.antechamber.prep -f prepi -o ${PDB}.cof.antechamber.frcmod
```
Tendo feito todos esses passos, execute o `tleap` e encontrará o seu sistema solvatado e neutralizado em dois arquivos, um `.prmtop` e um `.inpcrd` que contém respectivamente os parâmetros dos campos de força e as coordenadas de cada átomo do sistema. Use o script `amber_to_gmx.py` para criar os arquivos de entrada para o GROMACS, `${PDB}.gro` e `${PDB}.top`:
```bash
python ${path_to_script}/amber_to_gmx.py -p ${PDB}.com.solv.prmtop -c ${PDB}.com.solv.inpcrd
```
Copie os arquivos `.gro` e `.top` para a pasta `005.gmx_prep`.

### Equilibração pré-docking

Para criar os arquivos `.mdp` da minimização e da dinâmica, execute o script em `005.gmx_prep`:
```bash
bash ${path_to_script}/create_gromacs_mdps.dockprep.sh
```

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

*Importante!*
Os arquivos `.itp` precisam ser numerados a partir de 1 e não do número do `.gro` do sistema. Se for uma subestrutura da proteína, às vezes pode ser necessário renumerar usando um script de python específico.


#### Simulações de dinâmica molecular

Após colocar as cláusulas de restrição no arquivo `.top`, execute o GROMACS:
```bash
gmx grompp -f 01.minimization.mdp -p ${PDB}.top -c ${PDB}.gro -o 01.minimization.tpr
gmx mdrun -deffnm 01.minimization


gmx grompp -f 02.equil_nvt.mdp -p ${PDB}.top -c 01.minimization.gro -o 02.equil_nvt.tpr -r 01.minimization.gro
gmx mdrun -deffnm 02.equil_nvt

gmx grompp -f 03.minimization.mdp -p ${PDB}.top -c 02.equil_nvt.gro -o 03.minimization.tpr
gmx mdrun -deffnm 03.minimization
```

#### Extrair configuração final do resultado do GROMACS

O arquivo `03.minimization.gro` conterá a estrutura minimizada pronta para o docking. Precisamos, primeiramente, centralizar a proteína usando o comando:
```bash
gmx trjconv -f 03.minimization.gro -s 03.minimization.tpr -o centered.gro -pbc mol
```

Abra o arquivo `centered.gro` no UCSF Chimera ou UCSF ChimeraX, elimine íons desnecessários e todas as moléculas de água. Salve o complexo proteína e ligante como `${PDB}.ready.pdb` e transfira o arquivo para a pasta `06.dockprep`.

### Preparando os arquivos para o uso pelo DOCK6

Entre na pasta `06.dockprep` e abra sando o UCSF Chimera, abra `${PDB}.ready.pdb`. Primeiramente vá em:
```
Select > Residue > LIG
```
para selecionar a molécula de ligante. Apague-a com:
```
Actions > Atoms/Bonds > delete
```
Remanescerá a estrutura do receptor. Salve-a como `${PDB}.rec.pdb`. Apague todos os átomos de hidrogênio por meio da sequência:
```
Select > Chemistry > element > H
Actions > Atoms/Bonds > delete
```
e salve a estrutura restante como `${PDB}.rec.noH.pdb`. Reinicie o Chimera (`File > Close Session`) e reabra `${PDB}.ready.pdb`. Desta vez, selecione o ligante conforme as instruções acimae inverta a seleção:
```
Select > Invert (all models)
```
Ficará claro que a proteína está selecionada e o ligante não. Apague a proteína da forma exposta acima. O ligante original do complexo servirá de referência para as moléculas ancoradas ou criadas pelo DOCK6 e deve ser preparado como tal. Para isso:
```
Tools > Structure Editing > Dock Prep
```
Uma janela se abrirá com diversas opções que se aplicam a proteínas e ligantes. No caso do ligante, se aplicam somente as opções `Add hydrogens`, `Add charges` e `Write Mol2 file`. Clique em `OK` e outras janelas se abrirão. Se a sua molécula já tiver hidrogênios, nada acontecerá ao apertar `OK` na janela `Add Hydrogens for Dock Prep`. Caso não tenha, hidrogênios serão adicionados. A janela seguinte se refere à adição de cargas parciais atômicas. Escolha AM1-BCC e pressione `OK`. Uma nova janela aparecerá para confirmar a carga do ligante. Você deve saber qual é a carga da molécula e confira se é o que aparece sob `Net Charge`. Aperte `OK` e um cálculo quântico terá início para o cálculo das cargas.

Ao terminar, salve o arquivo como `${PDB}.lig.charged.mol2` e use o script em python para corrigir ajustes defeituosos:
```bash
python ${path_to_script}/fix_charges.py -m ${PDB}.lig.charged.mol2
```

Para preparar o receptor, abra `${PDB}.rec.pdb` e faça:
```
Tools > Structure Editing > Dock Prep
```
Aperte `OK` em todas as janelas e salve o arquivo como `${PDB}.rec.charged.mol2`.

### Determinando o sítio de ligação

Use o UCSF Chimera para abrir `${PDB}.rec.noH.pdb`. Mostre a superfície da proteína por meio de:
```
Actions > Surface > show
```
Salve a superfície por meio da seguinte sequência:
```
Tools > Structure Editing > Write DMS
```
Salve o arquivo como `${PDB}.rec.dms` na pasta `007.spheres`. Entre nessa pasta.

O sítio de ligação é representado por esferas. Para criá-las, temos que usar o programa `sphgen`, normalmente instalado junto ao `dock6`. Criar as esferas demanda criar um arquivo de input para o `sphgen`:
```bash
cat <<EOF > INSPH
${PDB}.rec.dms
R
X
0.0
4.0
1.4
${PDB}.rec.sph
EOF
```
A primeira linha do arquivo INSPH gerado pelo comando acima é a superfície usada como input para a criação das esferas.
`R` indica que as esferas serão criadas fora da superfície do receptor, `X` indica que todos os pontos da superfície serão explorados.
`0.0` é a distância em angstrom entre as esferas e a superfície; `4.0` é o raio máximo das esferas e `1.4` é o raio mínimo em angstroms.
A última linha de INSPH é o nome do arquivo contendo todas as esferas geradas pelo `sphgen`.
Para gerar as esferas:
```bash
sphgen -i INSPH -o OUTSPH
```
O arquivo `${PDB}.rec.sph` pode ser aberto no UCSF Chimera. Ele contém esferas ao redor de todo o receptor, o que não é interessante.
Precisamos selecionar as esferas que cobrem o ligante original, `${PDB}.lig.charged.mol2`.
Para isso usamos outro programa acessório instalado junto ao `dock6`, o `sphere_selector`:
```bash
sphere_selector ${PDB}.rec.sph ../006.dockprep/${PDB}.lig.charged.mol2 8.0
```
Observe que o caminho para a estrutura do ligante deve ser informada ao `sphere_selector` assim como uma distância em angstrom. No caso acima, todas as esferas em até 8.0 angstrom do ligante são selecionadas como parte do sítio de ligação.

### Gerando a caixa e o grid de energia potencial

Entre na pasta `008.grid` e crie o arquivo `showbox.in` com o comando abaixo:
Entre na pasta `008.grid` e crie o arquivo `showbox.in` com o comando abaixo:
```bash
cat <<EOF > showbox.in
Y
8.0
../007.spheres/selected_spheres.sph
1
${PDB}.box.pdb
EOF
```
A primeira linha do arquivo `showbox.in` significa que desejamos gerar uma caixa.
A segunda linha determina que cada lado da caixa deve ter comprimento de 8 angstrom.
A terceira linha diz que as esferas em `selected_spheres.sph` devem estar no centro da caixa.
A última linha determina o nome do arquivo de saída.
A caixa é gerada por meio do comando:
```bash
showbox < showbox.in
```

Em posse da caixa, `${PDB}.box.pdb`, podemos gerar o grid que será usado para calcular as interações entre as diversas poses e o receptor.
Crie o arquivo:
```bash
cat <<EOF > grid.in
compute_grids                             yes
grid_spacing                              0.4
output_molecule                           no
contact_score                             no
energy_score                              yes
energy_cutoff_distance                    9999
atom_model                                a
attractive_exponent                       6
repulsive_exponent                        12
distance_dielectric                       yes
dielectric_factor                         4
bump_filter                               yes
bump_overlap                              0.75
receptor_file                             ../006.dockprep/${PDB}.rec.charged.mol2
box_file                                  ${PDB}.box.pdb
vdw_definition_file                       ${caminho_para_a_pasta_do_dock6}/parameters/vdw_de_novo.defn
score_grid_prefix                         grid
EOF
```
O grid é criado pelo comando:
```bash
grid -i grid.in -o grid.out
```
o programa gerará três arquivos, `grid.out` que contém as informações da execução do programa, `grid.ngr` e `grid.bmp`, binários contendo o grid de energia.

### Próximos passos

Pronto! Agora você já pode fazer suas simulações com o `dock6`. Para _virtual screening_ ou _de novo_ design, veja tutorial específico.









