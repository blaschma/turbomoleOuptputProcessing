
<main>
<article id="content">
<header>
# Module `turbomoleOutputProcessing`
</header>
<section id="section-intro">
Package for processing turbomole Output such as mos files
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
    """
    Package for processing turbomole Output such as mos files
    """
    
    
    import numpy as np
    import math
    from scipy.sparse import coo_matrix
    from scipy.sparse.linalg import inv
    import scipy.sparse
    import re as r
    from scipy.sparse.linalg import inv
    from scipy.sparse import identity
    from scipy.linalg import eig
    from functools import partial
    from multiprocessing import Pool
    
    
    
    
    def outer_product(vector1, vector2):
            """
            Calculates out product
    
            Args:
                    param1 (np.array) : vector1
                    param2 (np.array) : vector2
    
            Returns:
                    np.ndarray
    
            """
            return np.outer(vector1,vector2)
    
    
    def eformat(f, prec, exp_digits):
            
        s = "%.*e"%(prec, f)
            #s ="hallo"
        mantissa, exp = s.split('e')
        # add 1 to digits as 1 is taken by sign +/-
        return "%sE%+0*d"%(mantissa, exp_digits+1, int(exp))
    
    
    def measure_asymmetry(matrix):  
            """ 
            gives a measure of asymmetry by calculatin max(|matrix-matrix^T|) (absvalue elementwise)
    
            Args:
                    param1 (np.ndarray) : Input Matrix
    
            Returns:
                    float
    
            """     
    
            return np.max(np.abs(matrix-np.transpose(matrix)))
    
     
    def measure_difference(matrix1, matrix2):
            """
            gives a measure of difference of two symmetric matrices (only upper triangular)
    
            Args:
                    param1 (np.ndarray) : matrix1
                    param2 (np.ndarray) : matrix2
    
            Returns: 
                    tuple of floats (min_error,max_error,avg_error,var_error)
            """
            diff = (np.abs(matrix1-matrix2))
            min_error = np.min(diff)
            print("min "  + str(min_error))
            max_error = np.max(diff)
            print("max "  + str(max_error))
            avg_error = np.mean(diff[np.triu_indices(diff.shape[0])])
            print("avg "  + str(avg_error))
            var_error = np.var(diff[np.triu_indices(diff.shape[0])])
            print("var "  + str(var_error))
            return min_error,max_error,avg_error,var_error
    
    
    
    
    
    def calculate_A(filename,prefix_length=2, eigenvalue_source = "mos", eigenvalue_path = "", eigenvalue_col = 2):
            """
            calculates the A matrix using mos file, reads eigenvalues from qpenergies (-&gt;DFT eigenvalues), input: mosfile and prefixlength (number of lines in the beginning), eigenvalue source (qpenergiesKS or qpenergiesGW), 
            if eigenvalue_path is specified you can give custom path to qpenergies and col to read
            returns matrix in scipy.sparse.csc_matrix and eigenvalue list
    
            Args:
                    param1 (string) : filename
                    param2 (int) : prefix_length (number of skipped lines in mos file)
                    param3 (string) : eigenvalue_source="mos" (choose eigenvalue source. If not changed: mos file. "qpenergiesKS" or "qpenergiesGW" : taken from ./data/qpenergies.dat)
                    param4 (string) : eigenvalue_path="" (if specified: eigenvalues are taken from qpenergies file with given path)
                    param5 (int) : eigenvalue_col (if eigenvalue_path is specified the eigenvalue_col in qpenergies is taken (KS=2, GW=3))
    
            Returns:
                    tuple A_i (scipy.sparse.csc_matrix) ,eigenvalue_list (list)
            """
            #length of each number
            n = 20
            level = 0
            C_vec = list()
            A_i = 0
            counter = 0
            eigenvalue_old = -1
            eigenvalue = 0
            beginning = True
            eigenvalue_list= list()
            #if eigenvalue source = qpEnergies
            if(eigenvalue_source == "qpenergiesKS" and eigenvalue_path == ""):
                    eigenvalue_list = read_qpenergies("qpenergies.dat", col = 1)
            elif(eigenvalue_source == "qpenergiesGW" and eigenvalue_path == ""):
                    eigenvalue_list = read_qpenergies("qpenergies.dat", col = 2)
            elif(eigenvalue_path != ""):
                    print("custom path ")
                    eigenvalue_list = read_qpenergies(eigenvalue_path, col = eigenvalue_col)
                    #print(eigenvalue_list)
    
    
            #open mos file and read linewise
            with open(filename) as f:
                    for line in f:
                            #skip prefix lines
                            #find level and calculate A(i)
                            if(counter&gt;prefix_length):
                                    index1 = (line.find("eigenvalue="))
                                    index2 = (line.find("nsaos="))
                                    #eigenvalue and nsaos found -&gt; new orbital
                                    if(index1 != -1 and index2 != -1):
                                            level += 1 
                                            
                                            #find eigenvalue of new orbital and store eigenvalue of current orbital in eigenvalue_old
                                            if((beginning == True and eigenvalue_source == "mos") and eigenvalue_path == ""):
                                                    eigenvalue_old = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))
                                                    eigenvalue = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))                                           
                                                    beginning = False
                                            elif(eigenvalue_source == "mos" and eigenvalue_path == ""):
                                                    #print("eigenvalue from mos")
                                                    eigenvalue_old = eigenvalue
                                                    eigenvalue_list.append(eigenvalue_old)
                                                    #if(level != 1128):
                                                    eigenvalue = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))
                                            #if eigenvalues are taken from qpenergies
                                            elif(eigenvalue_source == "qpenergiesKS" or eigenvalue_source == "qpenergiesGW" or eigenvalue_path != ""):
                                                    if(beginning == True):
                                                            beginning = False
                                                            pass
                                                    else:
                                                            #print("eigenvalue from eigenvaluelist")
                                                            eigenvalue_old = eigenvalue_list[level-2]
    
                                                                                                     
                                            #find nsaos of new orbital
                                            nsaos = (int(line[(index2+len("nsaos=")):len(line)]))
    
                                            #create empty A_i matrix
                                            if(isinstance(A_i,int)):
                                                    A_i = np.zeros((nsaos,nsaos))                                           
                                                    
    
                                            #calculate A matrix by adding A_i -&gt; processing of previous orbital             
                                            if(len(C_vec)&gt;0):                       
                                                    print("take " + str(eigenvalue_old))                                            
                                                    A_i += np.multiply(outer_product(C_vec,C_vec),eigenvalue_old)                                           
                                                    C_vec = list()                                  
                                            #everything finished (beginning of new orbital)
                                            continue
                                    
                                    
                                    line_split = [line[i:i+n] for i in range(0, len(line), n)]                                                      
                                    C_vec.extend([float(line_split[j].replace("D","E"))  for j in range(0,len(line_split)-1)][0:4])
                                    #print(len(C_vec))
                                    
                            counter += 1
                            #for testing
                            if(counter &gt; 300):
                                    #break
                                    pass
            #handle last mos
            if(eigenvalue_source == "qpenergiesKS" or eigenvalue_source == "qpenergiesGW"):
                    eigenvalue_old = eigenvalue_list[level-1]
                    print("lasteigen " + str(eigenvalue_old))
            
            A_i += np.multiply(outer_product(C_vec,C_vec),eigenvalue_old)
    
            A_i = scipy.sparse.csc_matrix(A_i, dtype = float)
            print("A_mat symmetric: " + str(np.allclose(A_i.toarray(), A_i.toarray(), 1e-04, 1e-04)))
            print("A_mat asymmetry measure " + str(measure_asymmetry(A_i.toarray())))
            print("------------------------")       
            return A_i,eigenvalue_list
    
    
    def read_packed_matrix(filename):
            """
            read packed symmetric matrix from file and creates matrix
            returns matrix in scipy.sparse.csc_matrix
            Args:
                    param1 (string) : filename
    
            Returns:
                    scipy.sparse.csc_matrix
    
            """     
            #matrix data
            data = []
            #matrix rows, cols
            i = []
            j = []
    
            counter =-1
            col = 0
            col_old = 0
            row = 0;
            #open smat_ao.dat and create sparse matrix
            with open(filename) as f:
                    for line in f:
                            #skip first line
                            if(counter == -1):
                                    counter+=1
                                    continue
                            #line_split = [i.split() for i in line]
                            line_split = r.split('\s+', line)
                            #print(counter)
                            #print(line_split[2])
    
                            matrix_entry = np.float(line_split[2])
                            #print(eformat(matrix_entry,100,5))
                            #print(line_split[2])
                            #print(eformat(np.longdouble(line_split[2]),100,5))
                            #print("-----")
                            #matrix_entry = round(matrix_entry,25)
                            matrix_enty = round(float(line_split[2]),24)
    
                            
    
                            #calculate row and col
                            
                            if(col == col_old+1):
                                    col_old = col
                                    col = 0;
                                    row += 1
                            #print("setting up row " + str(counter))
                            #print(row,col)
                            #skip zero elemts
                            if(matrix_entry != 0.0):
                                    data.append(matrix_entry)
                                    #print(matrix_entry)
                                    #print("------")
                                    i.append(col)
                                    j.append(row)
                                    #symmetrize matrix
                                    if(col != row):
                                            data.append(matrix_entry)
                                            i.append(row)
                                            j.append(col)
                                            pass
                            col += 1        
    
                            
                            counter+=1
                            #for testing
                            if(counter&gt;25):
                                    #break
                                    pass
            
            coo = coo_matrix((data, (i, j)), shape=(row+1, row+1))
            csc = scipy.sparse.csc_matrix(coo, dtype = float)       
            
            return(csc)
    
    
    
    
    def write_matrix_packed(matrix, filename="test"):
            """
            write symmetric scipy.sparse.csc_matrix in  in packed storage form
    
            Args:
                    param1 (scipi.sparse.csc_matrix) : input matrix
                    param2 (string) : filename
    
            Returns:
    
            """
            print("writing packed matrix")
            num_rows = matrix.shape[0]
            num_elements_to_write = (num_rows**2+num_rows)/2
            
    
            col = 0
            row = 0
            element_counter = 1
            f = open(filename, "w")
    
            line_beginning_spaces = ' ' * (12 - len(str(num_elements_to_write)))
            f.write(line_beginning_spaces + str(num_elements_to_write) + "      nmat\n")
    
            for r in range(0,num_rows):
                    #print("writing row " +str(r))
                    num_cols = r+1          
                    for c in range(0, num_cols):
                            matrix_element = matrix[r,c]
                            line_beginning_spaces = ' ' * (20 -len(str(int(element_counter))))
                            if(matrix_element&gt;=0):
                                    line_middle_spaces = ' ' * 16
                            else:
                                    line_middle_spaces = ' ' * 15
                            
                            f.write(line_beginning_spaces + str(int(element_counter)) + line_middle_spaces + eformat(matrix_element, 14,5) + "\n")                  
                            element_counter +=1
            f.close()
            print("writing done")
    
    
    def read_qpenergies(filename, col=1, skip_lines=1):
            """
            read qpenergies.dat and returns data of given col as list, skip_lines in beginning of file, convert to eV
            col = 0 qpenergies, col = 1 Kohn-Sham, 2 = GW
    
            Args:
                    param1 (string) : filename
                    param2 (int) : col=2 (column to read)
                    param3 (int) : skip_lines=1 (lines without data at beginning of qpenergiesfile)
    
            Returns:
                    list
            """
    
            har_to_ev = 27.21138602
            qpenergies = list()     
            datContent = [i.strip().split() for i in open(filename).readlines()[(skip_lines-1):]]
            #print(datContent)
            #datContent = np.transpose(datContent)
            #print(datContent)      
    
            for i in range(skip_lines, len(datContent)):
                    energy = float(datContent[i][col])/har_to_ev
                    #print("qpenergy " + str(energy))
                    qpenergies.append(energy)
                    pass    
            return qpenergies
    
    
    
    
    
    
    def write_mos_file(eigenvalues, eigenvectors, filename="mos_new.dat"):
            """
            write mos file, requires python3
    
            Args:
                    param1 (np.array) : eigenvalues
                    param2 (np.ndarray) : eigenvectors
                    param3 (string) : filename="mos_new.dat"
            """
            f = open(filename, "w")
            #header
            f.write("$scfmo    scfconv=7   format(4d20.14)\n")
            f.write("# SCF total energy is    -6459.0496515472 a.u.\n") 
            f.write("#\n")   
            for i in range(0, len(eigenvalues)):
                    print("eigenvalue " + str(eigenvalues[i]) + "\n")
                    first_string = ' ' * (6-len(str(i))) + str(i+1) + "  a      eigenvalue=" + eformat(eigenvalues[i], 14,2) + "   nsaos=" + str(len(eigenvalues))
                    f.write(first_string + "\n")
                    j = 0
                    while j&lt;len(eigenvalues):
                            for m in range(0,4):
                                    num = eigenvectors[m+j,i]
                                    #string_to_write = f"{num:+20.13E}".replace("E", "D")
                                    f.write(string_to_write)
                                    #f.write(eformat(eigenvectors[m+j,i], 14,2).replace("E", "D"))
                            f.write("\n")
                            j = j +4
                            #print("j " + str(j))
            f.write("$end")
            f.close()
    
    def read_mos_file(filename, skip_lines=1):
            """
            read mos file 
    
            Args:
                    param1 (string) : filename
                    param2 (int) : skip_lines=1 (lines to skip )
    
            Returns:
                    eigenvalue_list (list),eigenvector_list (np.ndarray)
    
    
            """
            n = 20
            level = 0
            C_vec = list()  
            counter = 0
            eigenvalue_old = -1
            eigenvalue = 0
            beginning = True
            eigenvalue_list= list() 
            eigenvector_list= -1    
    
    
            #open mos file and read linewise
            with open(filename) as f:
                    for line in f:
                            #skip prefix lines
                            #find level and calculate A(i)
                            if(counter&gt;skip_lines):
                                    index1 = (line.find("eigenvalue="))
                                    index2 = (line.find("nsaos="))
                                    #eigenvalue and nsaos found -&gt; new orbital
                                    if(index1 != -1 and index2 != -1):
                                            level += 1 
                                            #find nsaos of new orbital
                                            nsaos = (int(line[(index2+len("nsaos=")):len(line)]))   
    
                                            #find eigenvalue of new orbital and store eigenvalue of current orbital in eigenvalue_old
                                            if(beginning == True):
                                                    eigenvalue_old = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))
                                                    eigenvalue = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))
                                                    eigenvector_list = np.zeros((nsaos,nsaos),dtype=float)                                          
                                                    beginning = False
                                            else:
                                                    eigenvalue_old = eigenvalue     
                                                    
                                                    eigenvalue = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))
                                                                                                     
                                                                                                            
                                                    
    
                                            #calculate A matrix by adding A_i -&gt; processing of previous orbital             
                                            if(len(C_vec)&gt;0):                                                       
                                                    eigenvalue_list.append(eigenvalue_old)  
                                                    #print("level " + str(level))
                                                    eigenvector_list[:,(level-2)] = C_vec                                   
                                                    C_vec = list()                                  
                                            #everything finished (beginning of new orbital)
                                            continue
                                    
                                    
                                    line_split = [line[i:i+n] for i in range(0, len(line), n)]                                                      
                                    C_vec.extend([float(line_split[j].replace("D","E"))  for j in range(0,len(line_split)-1)][0:4])
                                    #print(len(C_vec))
                                    
                            counter += 1
                            
            #handle last mos
            eigenvalue_list.append(eigenvalue_old)  
            eigenvector_list[:,(nsaos-1)] = C_vec
            
            return eigenvalue_list,eigenvector_list
    
    
    
    def scalarproduct(index_to_check, para):        
            """
            helper function for trace_mo
    
    
            Args:
                    param1 (int) : index_to_check (index of mo)
                    param2 (tuple) : input parameter
            """
            ref_mos = para[0]
            input_mos = para[1]
            #abweichung von urspruenglicher Position
            tolerance = para[2]
            s_mat = para[3]
            most_promising = -1
            #print(ref_mos) 
            candidate_list = list()
            index_list = list()
            traced = False
            for i in range(0, ref_mos.shape[1]):    
                    if(tolerance != -1  and (i &lt; index_to_check+tolerance) and (i &gt; index_to_check-tolerance)):
                            scalar_product = list(np.dot(s_mat, ref_mos[:,i]).flat)
                            scalar_product = np.dot(np.transpose(input_mos[:,index_to_check]), scalar_product)
                            
                            if(np.abs(np.abs(float(scalar_product))-1)&lt;0.9):
                                    #print("scalar " + str(float(scalar_product)))  
                                    candidate_list.append(scalar_product)
                                    index_list.append(i)
                                    traced = True
                    elif(tolerance == -1):
                            scalar_product = list(np.dot(s_mat, ref_mos[:,i]).flat)
                            scalar_product = np.dot(np.transpose(input_mos[:,index_to_check]), scalar_product)
    
                            if(np.abs(np.abs(float(scalar_product))-1)&lt;0.9):
                                    traced = True
                                    candidate_list.append(scalar_product)
                                    index_list.append(i)
            #if they cannot be traced
            if(traced==False):
                    candidate_list.append(-1)
                    index_list.append(-1)
            #print("most_promising " + str(index_list))     
            if(traced == False):
                    #print("not found " + str(index_to_check))
                    pass
            traced = False
            most_promising = [x for _,x in sorted(zip(candidate_list,index_list))]
            most_promising = most_promising[-1]
            #print(str(index_to_check) + "  " + str(most_promising))
            return most_promising
            
    
                            
    def trace_mo(ref_mos, input_mos, s_mat_path, tolerance=-1, num_cpu = 8):        
            """
            traces mos from input_mos with reference to ref_mos (eg when order has changed) 
            calculates the scalarproduct of input mos with ref_mos and takes the highest match (close to 1)
    
            Args:
                    param1 (np.ndarray) : ref_mos
                    param2 (np.ndarray) : input_mos
                    param3 (string) : smat path
                    param4 (int) : tolerance = -1 (if specified a maximum shift of mo index (+-tolerance) is assumed -&gt; saves computation time)
                    param4 (int) : num_cpu (number of cores for calculation)
    
            Returns:
                    list (index with highest match)
            """
            input_mos_list = list()
            #prepare
            for i in range(0, ref_mos.shape[0]):
                    input_mos_list.append(input_mos[:,i])
            
            print("filling pool")
            p = Pool(num_cpu)       
            s_mat = read_packed_matrix(s_mat_path).todense()
            para = (ref_mos, input_mos, tolerance, s_mat)
    
            #all eigenvectors
    
            #index_to_check = range(0, input_mos.shape[0])
            index_to_check = np.arange(0,input_mos.shape[0],1)
            result = p.map(partial(scalarproduct, para=para), index_to_check)               
            print("done")
            return result
    
    
    
    
    
    def diag_F(f_mat_path, s_mat_path, eigenvalue_list = list()):
            """
            diagonalizes f mat (generalized), other eigenvalues can be used (eigenvalue_list). Solves Fx=l*S*x
            Args:
                    param1 (string) : filename of fmat
                    param2 (string) : filename of smat
                    param3 (list()) : eigenvalue_list (if specified eigenvalues are taken from eigenvalue_list)
    
            Returns:
                    fmat (np.ndarray), eigenvalues (np.array), eigenvectors (matrix)
    
            """
    
            print("read f")
            F_mat_file = read_packed_matrix(f_mat_path)
            print("read s")
            s_mat = read_packed_matrix(s_mat_path)
            s_mat_save = s_mat      
    
            print("diag F") 
            
    
            
            #Konfiguration 1 : Übereinstimmung
            #eigenvalues, eigenvectors = np.linalg.eigh(F_mat_file.todense())
    
            #Konfiguration 2 : keine Übereinstimmung (sollte eigentlich das gleiche machen wie np.linalg.eigh)
            #eigenvalues, eigenvectors = scipy.linalg.eigh(F_mat_file.todense())
    
            #Konfiguration 3: keine Übereinstimmung (generalierstes EW Problem)
            eigenvalues, eigenvectors = scipy.linalg.eigh(F_mat_file.todense(), s_mat.todense())
    
            eigenvectors = np.real(np.asmatrix(eigenvectors))
            #eigenvectors = np.transpose(eigenvectors)
            eigenvalues = np.real(eigenvalues)
            print(eigenvalues)
            #print("type random")
            #print(type(eigenvalues).__name__)
            #print(type(eigenvectors).__name__)
            print("diag F done")
    
            print("calc fmat ")
            #take eigenvalues from diagonalization or external eigenvalues (e.g from qpenergies)
            if(len(eigenvalue_list) == 0):
                    eigenvalues = np.diag(eigenvalues)
            else:           
                    eigenvalues = np.diag(eigenvalue_list)
    
            sc = s_mat * eigenvectors
            f_mat = eigenvalues * np.transpose(sc)
            f_mat = sc * f_mat
            print("calc fmat done")
            
    
    
            return f_mat,eigenvalues, eigenvectors
            
</details>
</section>
<section>
</section>
<section>
</section>
<section>
## Functions
<dl>
<dt id="turbomoleOutputProcessing.calculate_A"><code class="name flex">
<span>def <span class="ident">calculate_A</span></span>(<span>filename, prefix_length=2, eigenvalue_source='mos', eigenvalue_path='', eigenvalue_col=2)</span>
</code></dt>
<dd>
<div class="desc"><p>calculates the A matrix using mos file, reads eigenvalues from qpenergies (-&gt;DFT eigenvalues), input: mosfile and prefixlength (number of lines in the beginning), eigenvalue source (qpenergiesKS or qpenergiesGW),
if eigenvalue_path is specified you can give custom path to qpenergies and col to read
returns matrix in scipy.sparse.csc_matrix and eigenvalue list</p>
<h2 id="args">Args</h2>
<p>param1 (string) : filename
param2 (int) : prefix_length (number of skipped lines in mos file)
param3 (string) : eigenvalue_source="mos" (choose eigenvalue source. If not changed: mos file. "qpenergiesKS" or "qpenergiesGW" : taken from ./data/qpenergies.dat)
param4 (string) : eigenvalue_path="" (if specified: eigenvalues are taken from qpenergies file with given path)
param5 (int) : eigenvalue_col (if eigenvalue_path is specified the eigenvalue_col in qpenergies is taken (KS=2, GW=3))</p>
<h2 id="returns">Returns</h2>
<p>tuple A_i (scipy.sparse.csc_matrix) ,eigenvalue_list (list)</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def calculate_A(filename,prefix_length=2, eigenvalue_source = "mos", eigenvalue_path = "", eigenvalue_col = 2):
        """
        calculates the A matrix using mos file, reads eigenvalues from qpenergies (-&gt;DFT eigenvalues), input: mosfile and prefixlength (number of lines in the beginning), eigenvalue source (qpenergiesKS or qpenergiesGW), 
        if eigenvalue_path is specified you can give custom path to qpenergies and col to read
        returns matrix in scipy.sparse.csc_matrix and eigenvalue list

        Args:
                param1 (string) : filename
                param2 (int) : prefix_length (number of skipped lines in mos file)
                param3 (string) : eigenvalue_source="mos" (choose eigenvalue source. If not changed: mos file. "qpenergiesKS" or "qpenergiesGW" : taken from ./data/qpenergies.dat)
                param4 (string) : eigenvalue_path="" (if specified: eigenvalues are taken from qpenergies file with given path)
                param5 (int) : eigenvalue_col (if eigenvalue_path is specified the eigenvalue_col in qpenergies is taken (KS=2, GW=3))

        Returns:
                tuple A_i (scipy.sparse.csc_matrix) ,eigenvalue_list (list)
        """
        #length of each number
        n = 20
        level = 0
        C_vec = list()
        A_i = 0
        counter = 0
        eigenvalue_old = -1
        eigenvalue = 0
        beginning = True
        eigenvalue_list= list()
        #if eigenvalue source = qpEnergies
        if(eigenvalue_source == "qpenergiesKS" and eigenvalue_path == ""):
                eigenvalue_list = read_qpenergies("qpenergies.dat", col = 1)
        elif(eigenvalue_source == "qpenergiesGW" and eigenvalue_path == ""):
                eigenvalue_list = read_qpenergies("qpenergies.dat", col = 2)
        elif(eigenvalue_path != ""):
                print("custom path ")
                eigenvalue_list = read_qpenergies(eigenvalue_path, col = eigenvalue_col)
                #print(eigenvalue_list)


        #open mos file and read linewise
        with open(filename) as f:
                for line in f:
                        #skip prefix lines
                        #find level and calculate A(i)
                        if(counter&gt;prefix_length):
                                index1 = (line.find("eigenvalue="))
                                index2 = (line.find("nsaos="))
                                #eigenvalue and nsaos found -&gt; new orbital
                                if(index1 != -1 and index2 != -1):
                                        level += 1 
                                        
                                        #find eigenvalue of new orbital and store eigenvalue of current orbital in eigenvalue_old
                                        if((beginning == True and eigenvalue_source == "mos") and eigenvalue_path == ""):
                                                eigenvalue_old = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))
                                                eigenvalue = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))                                           
                                                beginning = False
                                        elif(eigenvalue_source == "mos" and eigenvalue_path == ""):
                                                #print("eigenvalue from mos")
                                                eigenvalue_old = eigenvalue
                                                eigenvalue_list.append(eigenvalue_old)
                                                #if(level != 1128):
                                                eigenvalue = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))
                                        #if eigenvalues are taken from qpenergies
                                        elif(eigenvalue_source == "qpenergiesKS" or eigenvalue_source == "qpenergiesGW" or eigenvalue_path != ""):
                                                if(beginning == True):
                                                        beginning = False
                                                        pass
                                                else:
                                                        #print("eigenvalue from eigenvaluelist")
                                                        eigenvalue_old = eigenvalue_list[level-2]

                                                                                                 
                                        #find nsaos of new orbital
                                        nsaos = (int(line[(index2+len("nsaos=")):len(line)]))

                                        #create empty A_i matrix
                                        if(isinstance(A_i,int)):
                                                A_i = np.zeros((nsaos,nsaos))                                           
                                                

                                        #calculate A matrix by adding A_i -&gt; processing of previous orbital             
                                        if(len(C_vec)&gt;0):                       
                                                print("take " + str(eigenvalue_old))                                            
                                                A_i += np.multiply(outer_product(C_vec,C_vec),eigenvalue_old)                                           
                                                C_vec = list()                                  
                                        #everything finished (beginning of new orbital)
                                        continue
                                
                                
                                line_split = [line[i:i+n] for i in range(0, len(line), n)]                                                      
                                C_vec.extend([float(line_split[j].replace("D","E"))  for j in range(0,len(line_split)-1)][0:4])
                                #print(len(C_vec))
                                
                        counter += 1
                        #for testing
                        if(counter &gt; 300):
                                #break
                                pass
        #handle last mos
        if(eigenvalue_source == "qpenergiesKS" or eigenvalue_source == "qpenergiesGW"):
                eigenvalue_old = eigenvalue_list[level-1]
                print("lasteigen " + str(eigenvalue_old))
        
        A_i += np.multiply(outer_product(C_vec,C_vec),eigenvalue_old)

        A_i = scipy.sparse.csc_matrix(A_i, dtype = float)
        print("A_mat symmetric: " + str(np.allclose(A_i.toarray(), A_i.toarray(), 1e-04, 1e-04)))
        print("A_mat asymmetry measure " + str(measure_asymmetry(A_i.toarray())))
        print("------------------------")       
        return A_i,eigenvalue_list</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.diag_F"><code class="name flex">
<span>def <span class="ident">diag_F</span></span>(<span>f_mat_path, s_mat_path, eigenvalue_list=[])</span>
</code></dt>
<dd>
<div class="desc"><p>diagonalizes f mat (generalized), other eigenvalues can be used (eigenvalue_list). Solves Fx=l<em>S</em>x</p>
<h2 id="args">Args</h2>
<p>param1 (string) : filename of fmat
param2 (string) : filename of smat
param3 (list()) : eigenvalue_list (if specified eigenvalues are taken from eigenvalue_list)</p>
<h2 id="returns">Returns</h2>
<p>fmat (np.ndarray), eigenvalues (np.array), eigenvectors (matrix)</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def diag_F(f_mat_path, s_mat_path, eigenvalue_list = list()):
        """
        diagonalizes f mat (generalized), other eigenvalues can be used (eigenvalue_list). Solves Fx=l*S*x
        Args:
                param1 (string) : filename of fmat
                param2 (string) : filename of smat
                param3 (list()) : eigenvalue_list (if specified eigenvalues are taken from eigenvalue_list)

        Returns:
                fmat (np.ndarray), eigenvalues (np.array), eigenvectors (matrix)

        """

        print("read f")
        F_mat_file = read_packed_matrix(f_mat_path)
        print("read s")
        s_mat = read_packed_matrix(s_mat_path)
        s_mat_save = s_mat      

        print("diag F") 
        

        
        #Konfiguration 1 : Übereinstimmung
        #eigenvalues, eigenvectors = np.linalg.eigh(F_mat_file.todense())

        #Konfiguration 2 : keine Übereinstimmung (sollte eigentlich das gleiche machen wie np.linalg.eigh)
        #eigenvalues, eigenvectors = scipy.linalg.eigh(F_mat_file.todense())

        #Konfiguration 3: keine Übereinstimmung (generalierstes EW Problem)
        eigenvalues, eigenvectors = scipy.linalg.eigh(F_mat_file.todense(), s_mat.todense())

        eigenvectors = np.real(np.asmatrix(eigenvectors))
        #eigenvectors = np.transpose(eigenvectors)
        eigenvalues = np.real(eigenvalues)
        print(eigenvalues)
        #print("type random")
        #print(type(eigenvalues).__name__)
        #print(type(eigenvectors).__name__)
        print("diag F done")

        print("calc fmat ")
        #take eigenvalues from diagonalization or external eigenvalues (e.g from qpenergies)
        if(len(eigenvalue_list) == 0):
                eigenvalues = np.diag(eigenvalues)
        else:           
                eigenvalues = np.diag(eigenvalue_list)

        sc = s_mat * eigenvectors
        f_mat = eigenvalues * np.transpose(sc)
        f_mat = sc * f_mat
        print("calc fmat done")
        


        return f_mat,eigenvalues, eigenvectors</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.eformat"><code class="name flex">
<span>def <span class="ident">eformat</span></span>(<span>f, prec, exp_digits)</span>
</code></dt>
<dd>
<div class="desc"></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def eformat(f, prec, exp_digits):
        
    s = "%.*e"%(prec, f)
        #s ="hallo"
    mantissa, exp = s.split('e')
    # add 1 to digits as 1 is taken by sign +/-
    return "%sE%+0*d"%(mantissa, exp_digits+1, int(exp))</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.measure_asymmetry"><code class="name flex">
<span>def <span class="ident">measure_asymmetry</span></span>(<span>matrix)</span>
</code></dt>
<dd>
<div class="desc"><p>gives a measure of asymmetry by calculatin max(|matrix-matrix^T|) (absvalue elementwise)</p>
<h2 id="args">Args</h2>
<p>param1 (np.ndarray) : Input Matrix</p>
<h2 id="returns">Returns</h2>
<p>float</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def measure_asymmetry(matrix):  
        """ 
        gives a measure of asymmetry by calculatin max(|matrix-matrix^T|) (absvalue elementwise)

        Args:
                param1 (np.ndarray) : Input Matrix

        Returns:
                float

        """     

        return np.max(np.abs(matrix-np.transpose(matrix)))</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.measure_difference"><code class="name flex">
<span>def <span class="ident">measure_difference</span></span>(<span>matrix1, matrix2)</span>
</code></dt>
<dd>
<div class="desc"><p>gives a measure of difference of two symmetric matrices (only upper triangular)</p>
<h2 id="args">Args</h2>
<p>param1 (np.ndarray) : matrix1
param2 (np.ndarray) : matrix2
Returns:
tuple of floats (min_error,max_error,avg_error,var_error)</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def measure_difference(matrix1, matrix2):
        """
        gives a measure of difference of two symmetric matrices (only upper triangular)

        Args:
                param1 (np.ndarray) : matrix1
                param2 (np.ndarray) : matrix2

        Returns: 
                tuple of floats (min_error,max_error,avg_error,var_error)
        """
        diff = (np.abs(matrix1-matrix2))
        min_error = np.min(diff)
        print("min "  + str(min_error))
        max_error = np.max(diff)
        print("max "  + str(max_error))
        avg_error = np.mean(diff[np.triu_indices(diff.shape[0])])
        print("avg "  + str(avg_error))
        var_error = np.var(diff[np.triu_indices(diff.shape[0])])
        print("var "  + str(var_error))
        return min_error,max_error,avg_error,var_error</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.outer_product"><code class="name flex">
<span>def <span class="ident">outer_product</span></span>(<span>vector1, vector2)</span>
</code></dt>
<dd>
<div class="desc"><p>Calculates out product</p>
<h2 id="args">Args</h2>
<p>param1 (np.array) : vector1
param2 (np.array) : vector2</p>
<h2 id="returns">Returns</h2>
<p>np.ndarray</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def outer_product(vector1, vector2):
        """
        Calculates out product

        Args:
                param1 (np.array) : vector1
                param2 (np.array) : vector2

        Returns:
                np.ndarray

        """
        return np.outer(vector1,vector2)</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.read_mos_file"><code class="name flex">
<span>def <span class="ident">read_mos_file</span></span>(<span>filename, skip_lines=1)</span>
</code></dt>
<dd>
<div class="desc"><p>read mos file </p>
<h2 id="args">Args</h2>
<p>param1 (string) : filename
param2 (int) : skip_lines=1 (lines to skip )</p>
<h2 id="returns">Returns</h2>
<p>eigenvalue_list (list),eigenvector_list (np.ndarray)</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def read_mos_file(filename, skip_lines=1):
        """
        read mos file 

        Args:
                param1 (string) : filename
                param2 (int) : skip_lines=1 (lines to skip )

        Returns:
                eigenvalue_list (list),eigenvector_list (np.ndarray)


        """
        n = 20
        level = 0
        C_vec = list()  
        counter = 0
        eigenvalue_old = -1
        eigenvalue = 0
        beginning = True
        eigenvalue_list= list() 
        eigenvector_list= -1    


        #open mos file and read linewise
        with open(filename) as f:
                for line in f:
                        #skip prefix lines
                        #find level and calculate A(i)
                        if(counter&gt;skip_lines):
                                index1 = (line.find("eigenvalue="))
                                index2 = (line.find("nsaos="))
                                #eigenvalue and nsaos found -&gt; new orbital
                                if(index1 != -1 and index2 != -1):
                                        level += 1 
                                        #find nsaos of new orbital
                                        nsaos = (int(line[(index2+len("nsaos=")):len(line)]))   

                                        #find eigenvalue of new orbital and store eigenvalue of current orbital in eigenvalue_old
                                        if(beginning == True):
                                                eigenvalue_old = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))
                                                eigenvalue = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))
                                                eigenvector_list = np.zeros((nsaos,nsaos),dtype=float)                                          
                                                beginning = False
                                        else:
                                                eigenvalue_old = eigenvalue     
                                                
                                                eigenvalue = float(line[(index1+len("eigenvalue=")):index2].replace("D","E"))
                                                                                                 
                                                                                                        
                                                

                                        #calculate A matrix by adding A_i -&gt; processing of previous orbital             
                                        if(len(C_vec)&gt;0):                                                       
                                                eigenvalue_list.append(eigenvalue_old)  
                                                #print("level " + str(level))
                                                eigenvector_list[:,(level-2)] = C_vec                                   
                                                C_vec = list()                                  
                                        #everything finished (beginning of new orbital)
                                        continue
                                
                                
                                line_split = [line[i:i+n] for i in range(0, len(line), n)]                                                      
                                C_vec.extend([float(line_split[j].replace("D","E"))  for j in range(0,len(line_split)-1)][0:4])
                                #print(len(C_vec))
                                
                        counter += 1
                        
        #handle last mos
        eigenvalue_list.append(eigenvalue_old)  
        eigenvector_list[:,(nsaos-1)] = C_vec
        
        return eigenvalue_list,eigenvector_list</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.read_packed_matrix"><code class="name flex">
<span>def <span class="ident">read_packed_matrix</span></span>(<span>filename)</span>
</code></dt>
<dd>
<div class="desc"><p>read packed symmetric matrix from file and creates matrix
returns matrix in scipy.sparse.csc_matrix</p>
<h2 id="args">Args</h2>
<p>param1 (string) : filename</p>
<h2 id="returns">Returns</h2>
<p>scipy.sparse.csc_matrix</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def read_packed_matrix(filename):
        """
        read packed symmetric matrix from file and creates matrix
        returns matrix in scipy.sparse.csc_matrix
        Args:
                param1 (string) : filename

        Returns:
                scipy.sparse.csc_matrix

        """     
        #matrix data
        data = []
        #matrix rows, cols
        i = []
        j = []

        counter =-1
        col = 0
        col_old = 0
        row = 0;
        #open smat_ao.dat and create sparse matrix
        with open(filename) as f:
                for line in f:
                        #skip first line
                        if(counter == -1):
                                counter+=1
                                continue
                        #line_split = [i.split() for i in line]
                        line_split = r.split('\s+', line)
                        #print(counter)
                        #print(line_split[2])

                        matrix_entry = np.float(line_split[2])
                        #print(eformat(matrix_entry,100,5))
                        #print(line_split[2])
                        #print(eformat(np.longdouble(line_split[2]),100,5))
                        #print("-----")
                        #matrix_entry = round(matrix_entry,25)
                        matrix_enty = round(float(line_split[2]),24)

                        

                        #calculate row and col
                        
                        if(col == col_old+1):
                                col_old = col
                                col = 0;
                                row += 1
                        #print("setting up row " + str(counter))
                        #print(row,col)
                        #skip zero elemts
                        if(matrix_entry != 0.0):
                                data.append(matrix_entry)
                                #print(matrix_entry)
                                #print("------")
                                i.append(col)
                                j.append(row)
                                #symmetrize matrix
                                if(col != row):
                                        data.append(matrix_entry)
                                        i.append(row)
                                        j.append(col)
                                        pass
                        col += 1        

                        
                        counter+=1
                        #for testing
                        if(counter&gt;25):
                                #break
                                pass
        
        coo = coo_matrix((data, (i, j)), shape=(row+1, row+1))
        csc = scipy.sparse.csc_matrix(coo, dtype = float)       
        
        return(csc)</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.read_qpenergies"><code class="name flex">
<span>def <span class="ident">read_qpenergies</span></span>(<span>filename, col=1, skip_lines=1)</span>
</code></dt>
<dd>
<div class="desc"><p>read qpenergies.dat and returns data of given col as list, skip_lines in beginning of file, convert to eV
col = 0 qpenergies, col = 1 Kohn-Sham, 2 = GW</p>
<h2 id="args">Args</h2>
<p>param1 (string) : filename
param2 (int) : col=2 (column to read)
param3 (int) : skip_lines=1 (lines without data at beginning of qpenergiesfile)</p>
<h2 id="returns">Returns</h2>
<p>list</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def read_qpenergies(filename, col=1, skip_lines=1):
        """
        read qpenergies.dat and returns data of given col as list, skip_lines in beginning of file, convert to eV
        col = 0 qpenergies, col = 1 Kohn-Sham, 2 = GW

        Args:
                param1 (string) : filename
                param2 (int) : col=2 (column to read)
                param3 (int) : skip_lines=1 (lines without data at beginning of qpenergiesfile)

        Returns:
                list
        """

        har_to_ev = 27.21138602
        qpenergies = list()     
        datContent = [i.strip().split() for i in open(filename).readlines()[(skip_lines-1):]]
        #print(datContent)
        #datContent = np.transpose(datContent)
        #print(datContent)      

        for i in range(skip_lines, len(datContent)):
                energy = float(datContent[i][col])/har_to_ev
                #print("qpenergy " + str(energy))
                qpenergies.append(energy)
                pass    
        return qpenergies</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.scalarproduct"><code class="name flex">
<span>def <span class="ident">scalarproduct</span></span>(<span>index_to_check, para)</span>
</code></dt>
<dd>
<div class="desc"><p>helper function for trace_mo</p>
<h2 id="args">Args</h2>
<p>param1 (int) : index_to_check (index of mo)
param2 (tuple) : input parameter</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def scalarproduct(index_to_check, para):        
        """
        helper function for trace_mo


        Args:
                param1 (int) : index_to_check (index of mo)
                param2 (tuple) : input parameter
        """
        ref_mos = para[0]
        input_mos = para[1]
        #abweichung von urspruenglicher Position
        tolerance = para[2]
        s_mat = para[3]
        most_promising = -1
        #print(ref_mos) 
        candidate_list = list()
        index_list = list()
        traced = False
        for i in range(0, ref_mos.shape[1]):    
                if(tolerance != -1  and (i &lt; index_to_check+tolerance) and (i &gt; index_to_check-tolerance)):
                        scalar_product = list(np.dot(s_mat, ref_mos[:,i]).flat)
                        scalar_product = np.dot(np.transpose(input_mos[:,index_to_check]), scalar_product)
                        
                        if(np.abs(np.abs(float(scalar_product))-1)&lt;0.9):
                                #print("scalar " + str(float(scalar_product)))  
                                candidate_list.append(scalar_product)
                                index_list.append(i)
                                traced = True
                elif(tolerance == -1):
                        scalar_product = list(np.dot(s_mat, ref_mos[:,i]).flat)
                        scalar_product = np.dot(np.transpose(input_mos[:,index_to_check]), scalar_product)

                        if(np.abs(np.abs(float(scalar_product))-1)&lt;0.9):
                                traced = True
                                candidate_list.append(scalar_product)
                                index_list.append(i)
        #if they cannot be traced
        if(traced==False):
                candidate_list.append(-1)
                index_list.append(-1)
        #print("most_promising " + str(index_list))     
        if(traced == False):
                #print("not found " + str(index_to_check))
                pass
        traced = False
        most_promising = [x for _,x in sorted(zip(candidate_list,index_list))]
        most_promising = most_promising[-1]
        #print(str(index_to_check) + "  " + str(most_promising))
        return most_promising</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.trace_mo"><code class="name flex">
<span>def <span class="ident">trace_mo</span></span>(<span>ref_mos, input_mos, s_mat_path, tolerance=-1, num_cpu=8)</span>
</code></dt>
<dd>
<div class="desc"><p>traces mos from input_mos with reference to ref_mos (eg when order has changed)
calculates the scalarproduct of input mos with ref_mos and takes the highest match (close to 1)</p>
<h2 id="args">Args</h2>
<p>param1 (np.ndarray) : ref_mos
param2 (np.ndarray) : input_mos
param3 (string) : smat path
param4 (int) : tolerance = -1 (if specified a maximum shift of mo index (+-tolerance) is assumed -&gt; saves computation time)
param4 (int) : num_cpu (number of cores for calculation)</p>
<h2 id="returns">Returns</h2>
<p>list (index with highest match)</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def trace_mo(ref_mos, input_mos, s_mat_path, tolerance=-1, num_cpu = 8):        
        """
        traces mos from input_mos with reference to ref_mos (eg when order has changed) 
        calculates the scalarproduct of input mos with ref_mos and takes the highest match (close to 1)

        Args:
                param1 (np.ndarray) : ref_mos
                param2 (np.ndarray) : input_mos
                param3 (string) : smat path
                param4 (int) : tolerance = -1 (if specified a maximum shift of mo index (+-tolerance) is assumed -&gt; saves computation time)
                param4 (int) : num_cpu (number of cores for calculation)

        Returns:
                list (index with highest match)
        """
        input_mos_list = list()
        #prepare
        for i in range(0, ref_mos.shape[0]):
                input_mos_list.append(input_mos[:,i])
        
        print("filling pool")
        p = Pool(num_cpu)       
        s_mat = read_packed_matrix(s_mat_path).todense()
        para = (ref_mos, input_mos, tolerance, s_mat)

        #all eigenvectors

        #index_to_check = range(0, input_mos.shape[0])
        index_to_check = np.arange(0,input_mos.shape[0],1)
        result = p.map(partial(scalarproduct, para=para), index_to_check)               
        print("done")
        return result</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.write_matrix_packed"><code class="name flex">
<span>def <span class="ident">write_matrix_packed</span></span>(<span>matrix, filename='test')</span>
</code></dt>
<dd>
<div class="desc"><p>write symmetric scipy.sparse.csc_matrix in
in packed storage form</p>
<h2 id="args">Args</h2>
<p>param1 (scipi.sparse.csc_matrix) : input matrix
param2 (string) : filename
Returns:</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def write_matrix_packed(matrix, filename="test"):
        """
        write symmetric scipy.sparse.csc_matrix in  in packed storage form

        Args:
                param1 (scipi.sparse.csc_matrix) : input matrix
                param2 (string) : filename

        Returns:

        """
        print("writing packed matrix")
        num_rows = matrix.shape[0]
        num_elements_to_write = (num_rows**2+num_rows)/2
        

        col = 0
        row = 0
        element_counter = 1
        f = open(filename, "w")

        line_beginning_spaces = ' ' * (12 - len(str(num_elements_to_write)))
        f.write(line_beginning_spaces + str(num_elements_to_write) + "      nmat\n")

        for r in range(0,num_rows):
                #print("writing row " +str(r))
                num_cols = r+1          
                for c in range(0, num_cols):
                        matrix_element = matrix[r,c]
                        line_beginning_spaces = ' ' * (20 -len(str(int(element_counter))))
                        if(matrix_element&gt;=0):
                                line_middle_spaces = ' ' * 16
                        else:
                                line_middle_spaces = ' ' * 15
                        
                        f.write(line_beginning_spaces + str(int(element_counter)) + line_middle_spaces + eformat(matrix_element, 14,5) + "\n")                  
                        element_counter +=1
        f.close()
        print("writing done")</code></pre>
</details>
</dd>
<dt id="turbomoleOutputProcessing.write_mos_file"><code class="name flex">
<span>def <span class="ident">write_mos_file</span></span>(<span>eigenvalues, eigenvectors, filename='mos_new.dat')</span>
</code></dt>
<dd>
<div class="desc"><p>write mos file, requires python3</p>
<h2 id="args">Args</h2>
<p>param1 (np.array) : eigenvalues
param2 (np.ndarray) : eigenvectors
param3 (string) : filename="mos_new.dat"</p></div>
<details class="source">
<summary>
<span>Expand source code</span>
</summary>
<pre><code class="python">def write_mos_file(eigenvalues, eigenvectors, filename="mos_new.dat"):
        """
        write mos file, requires python3

        Args:
                param1 (np.array) : eigenvalues
                param2 (np.ndarray) : eigenvectors
                param3 (string) : filename="mos_new.dat"
        """
        f = open(filename, "w")
        #header
        f.write("$scfmo    scfconv=7   format(4d20.14)\n")
        f.write("# SCF total energy is    -6459.0496515472 a.u.\n") 
        f.write("#\n")   
        for i in range(0, len(eigenvalues)):
                print("eigenvalue " + str(eigenvalues[i]) + "\n")
                first_string = ' ' * (6-len(str(i))) + str(i+1) + "  a      eigenvalue=" + eformat(eigenvalues[i], 14,2) + "   nsaos=" + str(len(eigenvalues))
                f.write(first_string + "\n")
                j = 0
                while j&lt;len(eigenvalues):
                        for m in range(0,4):
                                num = eigenvectors[m+j,i]
                                #string_to_write = f"{num:+20.13E}".replace("E", "D")
                                f.write(string_to_write)
                                #f.write(eformat(eigenvectors[m+j,i], 14,2).replace("E", "D"))
                        f.write("\n")
                        j = j +4
                        #print("j " + str(j))
        f.write("$end")
        f.close()</code></pre>
</details>
</dd>
</dl>
</section>
<section>
</section>
</article>
<nav id="sidebar">
# Index
<div class="toc">
<ul></ul>
</div>

- <h3>[Functions](#header-functions)</h3>
        
    - `<a title="turbomoleOutputProcessing.calculate_A" href="#turbomoleOutputProcessing.calculate_A">calculate_A</a>`
    - `<a title="turbomoleOutputProcessing.diag_F" href="#turbomoleOutputProcessing.diag_F">diag_F</a>`
    - `<a title="turbomoleOutputProcessing.eformat" href="#turbomoleOutputProcessing.eformat">eformat</a>`
    - `<a title="turbomoleOutputProcessing.measure_asymmetry" href="#turbomoleOutputProcessing.measure_asymmetry">measure_asymmetry</a>`
    - `<a title="turbomoleOutputProcessing.measure_difference" href="#turbomoleOutputProcessing.measure_difference">measure_difference</a>`
    - `<a title="turbomoleOutputProcessing.outer_product" href="#turbomoleOutputProcessing.outer_product">outer_product</a>`
    - `<a title="turbomoleOutputProcessing.read_mos_file" href="#turbomoleOutputProcessing.read_mos_file">read_mos_file</a>`
    - `<a title="turbomoleOutputProcessing.read_packed_matrix" href="#turbomoleOutputProcessing.read_packed_matrix">read_packed_matrix</a>`
    - `<a title="turbomoleOutputProcessing.read_qpenergies" href="#turbomoleOutputProcessing.read_qpenergies">read_qpenergies</a>`
    - `<a title="turbomoleOutputProcessing.scalarproduct" href="#turbomoleOutputProcessing.scalarproduct">scalarproduct</a>`
    - `<a title="turbomoleOutputProcessing.trace_mo" href="#turbomoleOutputProcessing.trace_mo">trace_mo</a>`
    - `<a title="turbomoleOutputProcessing.write_matrix_packed" href="#turbomoleOutputProcessing.write_matrix_packed">write_matrix_packed</a>`
    - `<a title="turbomoleOutputProcessing.write_mos_file" href="#turbomoleOutputProcessing.write_mos_file">write_mos_file</a>`
    


</nav>
</main>
<footer id="footer">
Generated by [<cite>pdoc</cite> 0.8.4](https://pdoc3.github.io/pdoc).
</footer>
