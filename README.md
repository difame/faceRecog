# faceRecog
# -*- coding: utf-8 -*-
from io import BytesIO
import sys, os

class PspParser:
    begin_source = 1
    begin_var = 2
    end_mark = 3

    def fileToString(self, p_file):
        with open(p_file, 'r') as file:
            return file.read()

    def seeToken(self, c):
        if  len(self.source) <= c + 1:
            return 0, None
        if  c + 3 <= len(self.source) and \
            self.source[c+0] == '<'   and \
            self.source[c+1] == '%'   and \
            self.source[c+2] == '=' :
            return 3, self.begin_var
        if  c + 2 <= len(self.source) and \
            self.source[c+0] == '<'   and \
            self.source[c+1] == '%' :
            return 2, self.begin_source
        return 1, self.source[c]

    def findHtmlEnd(self, c):
        while c < len(self.source) and self.source[c] != "\n":
            if c + 2 <= len(self.source) and \
                self.source[c + 0] == '<' and \
                self.source[c + 1] == '%':
                return c, False
            c += 1
        return c, True
    def findKeyword2(self, c, key):
        while c + 1 < len(self.source) and (self.source[c] != key[0] or self.source[c+1] != key[1]):
            c += 1
        return c

    def parse(self, psp_name, py_name):
        text = self.fileToString(psp_name)

        with open(py_name, 'w') as gen:
            self.source = list(text)
            c = 0
            gen.write("f = BytesIO()\n")
            while True:
                sz, tok = self.seeToken(c)
                if sz == 0:
                    break;
                elif tok == self.begin_source:
                    c += sz
                    n = self.findKeyword2(c, "%>")
                    gen.write("#python source begin\n")
                    gen.write(''.join(self.source[c:n]))
                    gen.write("#python source end\n")
                    c = (n + 2)
                elif tok == self.begin_var:
                    c += sz
                    n = self.findKeyword2(c, "%>")
                    gen.write("\tf.write(" + ''.join(self.source[c:n]) + ")\n")
                    c = (n + 2)
                else:
                    # <%, line feed 나올떄 까지  읽기
                    n, is_line_end = self.findHtmlEnd(c)
                    stmt = ''.join(self.source[c: n]).replace('\"', '\\\"')
                    if is_line_end:
                        gen.write("\tf.write(\"" + stmt + "\\n\")\n")
                        c = n + 1
                    else :
                        gen.write("\tf.write(\"" + stmt + "\")\n")
                        c = n
        print("generate {}".format(py_name))

if __name__ == '__main__':
    p = PspParser()
    if len(sys.argv) <= 1:
        print('usage: psp to pytheon code_generator')
        print('        python -g psp.py file[.psp]')
        print('            file.psp ==> file.py ')
        print('            ex) html_code<% python code %>  html_code <%=var%> html_code ')

    for f in sys.argv[1:]:
        psp_name = f.replace(".psp", "") + ".psp"
        py_name  = f.replace(".psp", "") + ".py"
        if os.path.isfile(psp_name):
            p.parse(psp_name, py_name)
