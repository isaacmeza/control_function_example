
---
# "Control Function Methods"
---


In this note we study control function (CF) methods. We use data from a RCT in MCLC @sadka, to study the effect of statistical information ($x_j$) provided to dismissed workers *when the worker is present* ($t_j$)  on settlement rates ($y_j$).



The calculator arm ($x_j$) is random, but we might be concerned with the endogeneity of employee presence ($t_j$).  While this is fundamentally an external validity issue (since the calculator is random conditional on the plaintiff's presence), it is relevant for how we interpret the null treatment effect when the employee is not present. To address any concern with the endoegeneity of the plaintiff's presence, we use control function approaches. We use time of hearing ($w_j$) as instrument. 


```s
import delimited "https://raw.githubusercontent.com/isaacmeza/control_function_example/main/DB/cf_pactor_sett.csv", clear  case(preserve)
```

To motivate the use of control function methods, we contrast it with the usual 2sls estimator. We cluster the errors at the level of randomization and use basic controls.

```s
ivregress 2sls seconcilio calc d_anio* d_num* d_junta* phase (p_actor calc_x_pactor = i.time_hr##i.calc), vce(cluster cluster_v)   
```


We are interested in the coefficient of the interaction **calc_x_pactor**. We find an effect of <<dd_display: %9.3f _b[calc_x_pactor]>>, however we cannot reject the null of no significance (we have a std. error of <<dd_display: %9.2f _se[calc_x_pactor]>>). The advantage of CF methods is to achieve greater efficiency exploting the binary nature of the EEV, in this case 'presence of the employee'.


### Constrained model - Constant Coefficients

The model of interest is
$$ y_j = x_j\beta +\delta t_j+\epsilon_j$$

where $t_j$ is a binary-treatment variable generated by an unobservable latent model:
$$t_j = 1[w_j\gamma + u_j>0]$$
and $\epsilon$, $u$ are bivariate normal with mean zero and covariance matrix

$$ \begin{bmatrix} \sigma^2 
& \rho\sigma 
\end{bmatrix}$$ 
$$ \begin{bmatrix}\rho\sigma & 1 
\end{bmatrix}$$ 


We also allow interactions between $x_j$ and the treatment $t_j$. 





#### Two-step CF

For the two-step control function procedure, probit estimates of the (EEV) 
$$\Pr(T_j=1|w_j) = \Phi(w_j\gamma)$$
are obtained in the first stage.
From these estimates we obtain "generalized residuals" 
$$r_j = t_j(\frac{\phi(w_j\hat{\gamma})}{\Phi(w_j\hat{\gamma})})+(1-t_j)(-\frac{\phi(w_j\hat{\gamma})}{1-\Phi(w_j\hat{\gamma})})$$

then
$$\mathbb{E}[y_j|t_j,x_j,w_j]  = x_j\beta +\delta t_j+\rho\sigma r_j$$
$$\operatorname{Var}(y_j|t_j,x_j,w_j) = \sigma^2(1-\rho^2(r_j(r_j+w_j\hat{\gamma})))$$

the two-step estimates of $\beta$ and $\delta$ are obtained by augmenting the regression with the generalized residuals, thus the regressors become $[x \quad t \quad r]$.
The t statistic on $r_j$, made robust to heteroskedasticity, is a valid test of the null that $t_j$ is exogenous.


We "manually" implement the previous approach as follows:

1) Estimate first stage using probit
```s
*Probit (FS)
probit p_actor calc i.time_hr d_anio* d_num* d_junta* phase
```

 obtain generalized residual to form the augmented regression in the second stage
```s
*Linear predictions
cap drop xb
predict xb, xb
*Generalized residuals
cap drop gen_resid_pr
gen gen_resid_pr = cond(p_actor == 1, normalden(xb)/normal(xb), -normalden(xb)/(1-normal(xb)))	
```

2) Run the second stage. We use bootstrap to estimate standard errors, since estimation is done in two stages. 

```s
*Second stage
reg seconcilio calc p_actor calc_x_pactor gen_resid_p d_anio* d_num* d_junta* phase , vce(bootstrap, cluster(cluster_v) rep(100) bca)
```



Note that by exploting the binary nature of the endogenous variable, we achieve more efficiency. The coefficient of the interaction is <<dd_display: %9.3f _b[calc_x_pactor]>>, which is smaller than the 2sls IV estimate, and we get significance at the <<dd_display: %9.3f 200*ttail(e(N)-e(df_m), abs(_b[calc_x_pactor]/_se[calc_x_pactor]))>>% level. 
Moreover, we can easily reject the null of exogeneity of the employee present (*p_actor*) by looking at the p-value of the variable *gen_resid_pr* (<<dd_display: %9.3f 200*ttail(e(N)-e(df_m), abs(_b[gen_resid_p]/_se[gen_resid_p]))>>).


The previous estimation can be also done with the `etregress` command with the `two-step` option

An advantage using this command is the analytical derivation of the covariance matrix.

A consistent estimate of the regression distrubance variance is obtained by
$$ \hat{\sigma^2} = \frac{e^\prime e+\beta_r^2\sum_{j}r_j(r_j+w_j\hat{\gamma})}{N}$$
with $\beta_r$ the parameter estimate of the generalized residual. The two-step estimate of $\rho$ is then
$$\hat{\rho} = \frac{\beta_r}{\hat{\sigma}}$$

Finally, let $A = [x \quad  t \quad  r]$ and $D = diag(1-\hat{\rho}^{2} r_{j}(r_{j}+w_{j}\hat{\gamma}))$. The covariance matrix is 
$$ V_{twostep} = \hat{\sigma}^2(A^\prime A)^{-1}(A^\prime DA + Q)(A^\prime A)^{-1}$$
with 
$$Q = \hat{\rho}^2(A^\prime DA)V_p(A^\prime DA)$$
and $V_p$ the covariance estimate from the probit first-stage.

```s
etregress seconcilio i.calc##i.p_actor d_anio* d_num* d_junta* phase, treat(p_actor = i.time_hr calc d_anio* d_num* d_junta* phase )  twostep 
```


The point estimate is the same as in the previous method, but the std. error (<<dd_display: %9.3f _se[1.calc#1.p_actor]>>) is smaller.


#### GMM

Previously we did not exploit clustering of the standard errors. To alleviate this a one-step CF GMM estimator can be implemented. 

We form the parameter vector $\theta = (\beta^\prime,  \delta, \gamma^\prime, \operatorname{atanh}\rho, \ln\sigma)^\prime$.

We have three separate error functions for the EEV, the outcome mean, and the outcome variance:
$$u_t(t_j,w_j,\theta) =  t_j(\frac{\phi(w_j{\gamma})}{\Phi(w_j{\gamma})})+(1-t_j)(-\frac{\phi(w_j{\gamma})}{1-\Phi(w_j{\gamma})})$$
$$u_m(y_j,t_j,x_j,w_j,\theta) = y_j-x_j\beta -\delta t_j-\rho\sigma u_{t,j} $$
$$u_v(y_j,t_j,x_j,w_j,\theta) = u^2_{m,j} -\sigma^2 [1-\rho^2(u_{t,j}(u_{t,j}+w_j\gamma))]$$

Let $z_j = (x_j,t_j,r_j)$ , $Z_j = diag(z_j, w_j, 1)$, and
$$s_j(y_j, t_j, x_j,w_j,\theta) = Z_{j}^{\prime}(u_{m,j},u_{t,j},u_{v,j})^{\prime} $$
The CF estimator $\hat{\theta}$ is the value of $\theta$ thats satisfies the sample-moment conditions.
$$0 = \frac{1}{N}\sum_{j}s_j(y_j, t_j, x_j,w_j,\theta)$$


The variance estimator is
$$\hat{V} = \frac{1}{N}GSG^{\prime}$$
with
$$ G = \left((1/N)\sum_{j} \frac{\partial s_j(y_j, t_j, x_j,w_j,\theta) }{\partial \theta}\right)^{-1}$$
$$ S = (1/N)\sum_{j} s_j(y_j, t_j, x_j,w_j,\theta)s_j(y_j, t_j, x_j,w_j,\theta)^{\prime}$$


This is done using the option `cfunction` in the `etregress` command 

```s
etregress seconcilio i.calc##i.p_actor d_anio* d_num* d_junta* phase, treat(p_actor = i.time_hr calc d_anio* d_num* d_junta* phase )  cfunction vce(cluster cluster_v) 
```

Adittionally, the output displays a Wald test for the null $H_0 : \rho = 0$, which serves as a null to test the exogeneity of $t_j$.

#### Maximum-Likelihood

Finally the command `etregress` also estimates via maximum-likelihood the parameters of interest.

The log-likelihood function for observation $j$ is given by
$$ ln L_j = t_j\left(ln \Phi\left\lbrace\frac{w_j\gamma+(y_j-x_j\beta-\delta)\rho/\sigma}{\sqrt{1-\rho^2}}\right\rbrace-\frac{1}{2}\left(\frac{y_j-x_j\beta-\delta}{\sigma}\right)^2-ln(\sqrt{2\pi}\sigma)\right)$$
$$\quad\quad + (1-t_j)\left(ln \Phi\left\lbrace\frac{-w_j\gamma+(y_j-x_j\beta)\rho/\sigma}{\sqrt{1-\rho^2}}\right\rbrace-\frac{1}{2}\left(\frac{y_j-x_j\beta}{\sigma}\right)^2-ln(\sqrt{2\pi}\sigma)\right)$$

```s
etregress seconcilio i.calc##i.p_actor d_anio* d_num* d_junta* phase , treat(p_actor = i.time_hr calc d_anio* d_num* d_junta* phase) vce(cluster cluster_v)   
```

      

As suggested by @Wooldridge, when the EEV $t_j$ interacts with $x_j$ there are other natural choices for IVs: Use the interactions $\Phi(w_j\hat{\gamma})x_j$ where $\Phi(w_j\hat{\gamma})$ are the predicted probabilities of $t_j$. This IV estimator has a theoretical advantage over the CF estimator, at least if one assumes the linear model with constant coefficients is the correct specification: The IV estimator is generally consistent even if the probit model is misspecified.




### General potential outcome model - Correlated Random Coefficient

The previous setup allows the endogenous explanatory variable or variables to appear linearly or nonlinearly and to interact with observed covariates. This may be sufficient for some applications, but one may also want to allow the effect of the EEV to depend on unobservables.


The recent literature on instrumental variables (IV) features models in which agents sort into treatment status on the basis of gains from treatment as well as on baseline-pretreatment levels. Components of the gains known to the agents and acted on by them may not be known by the observing economist. Such models are called **correlated random coefficient models** (CRC). CRC models allow for heterogeneous treatment effects combined with self-selection into treatment ??? provided that there are suitable instrumental variables for treatment assignment.
 
 
 We consider the model 
 
$$ y_{0j} = x_j\beta_0 +\epsilon_{0j}$$
$$ y_{1j} = x_j\beta_1 +\epsilon_{1j}$$

where $t_j$ is a binary-treatment variable generated by an unobservable latent model:
$$t_j = 1[w_j\gamma + u_j>0]$$
and
$$ y_{j} = t_jy_{1j}+(1-t_j)y_{0j}$$

We now rely on a trivariate normality assumption between the error terms of the potential outcomes and the error term of the EEV $(\epsilon_{0j}, \epsilon_{1j}, u_j)$, whose covariance structure is

$$ \begin{bmatrix} \sigma^2_0 & \sigma_{01} 
& \rho_0\sigma_0 
\end{bmatrix}$$ 
$$ \begin{bmatrix}\sigma_{01}  & \sigma^2_1 & \rho_1\sigma_1
\end{bmatrix}$$ 
$$ \begin{bmatrix}\rho_0\sigma_0   & \rho_1\sigma_1 & 1
\end{bmatrix}$$ 

By allowing group-specific variance and correlation, we allow for no-observable heterogeneity, and heterogeneous reaction function of $y_0$ and $y_1$ to $x$.


As in the constant-coefficient case, we can follow a two-step procedure, GMM, or ML.

#### Two-step 

The endogenous switching model is

$$ \mathbb{E}(y_j | t_j,x_j,w_j) = t_j(x_j\beta_1+\rho_1\sigma_1u_{t,j})+(1-t_j)(x_j\beta_0+\rho_0\sigma_0u_{t,j})$$
$$\quad\quad = $$
 


```s
su calc, meanonly
*Center interaction
gen calc_c_pactor = p_actor*(calc-`r(mean)')

reg seconcilio calc calc_c_pactor i.p_actor##c.gen_resid_pr  d_anio* d_num* d_junta* phase , vce(bootstrap, cluster(cluster_v) rep(100) bca)  
```
Compared with the constant-coefficient case, I have added the interaction `i.p_actor##c.gen_resid_pr`. The interaction term accounts for the random coefficient on $t_j$. (See eqs. (22)-(26) @Wooldridge)

Centering *calc* about the sample average ensures that the coefficient on *p_actor* is the average effect.

Finally, a test of joint significance of $()$ is valid without adjusting the standard errors. The joint test is a test of the null hypothesis that $t_j$ is exogenous.

```s
test _b[gen_resid_pr]=_b[1.p_actor#gen_resid_pr]=0
```


Alternatively we can use the command `ivtreatreg` to achieve the same result. The option `heckit` specifies a potential outcome model with a probit model for the EEV. 

$$ \mathbb{E}(y_j | t_j,x_j,w_j) = t_j(x_j\beta_1+\rho_1\sigma_1u_{t,j})+(1-t_j)(x_j\beta_0+\rho_0\sigma_0u_{t,j})$$
$$\quad\quad = $$

where is the ATE, ??1and??0are the correlations between the two potential outcomes'errors and the treatment's error



```s
*Generate dummies of time of hearing (since ivtreatreg does not allow factor variables)
tab time_hr, gen(time_hr)
drop time_hr1

ivtreatreg seconcilio p_actor calc d_anio* d_num* d_junta* phase, model(heckit) iv(time_hr2-time_hr8) hetero(calc) vce(robust)
```

The test for selection on un-observables is slightly different in this case $H_0 : =0$

```s
test _b[_wL1]=_b[_wL0]=0
```


#### GMM

The parameter vector is $\theta = (\beta_0^\prime, \beta_1^\prime,\gamma^\prime, \operatorname{atanh}\rho_0, \ln\sigma_0 ,\operatorname{atanh}\rho_1, \ln\sigma_1)$. Given the following identities

$$ \mathbb{E}(y_j | t_j,x_j,w_j) = t_j(x_j\beta_1+\rho_1\sigma_1u_{t,j})+(1-t_j)(x_j\beta_0+\rho_0\sigma_0u_{t,j})$$
$$\operatorname{Var}(y_j|t_j=0,x_j,w_j) = \sigma_0^2(1-\rho_0^2(u_{t,j}(u_{t,j}+w_j\gamma)))$$
$$\operatorname{Var}(y_j|t_j=1,x_j,w_j) = \sigma_1^2(1-\rho_1^2(u_{t,j}(u_{t,j}+w_j\gamma)))$$

we can obtain error functions for the outcome means and variances, and analogous to GMM in the constant-coefficient case, the estimator $\hat{\theta}$ is the value of $\theta$ that satisfies the sample moment condition.


We further specify the option `poutcomes` in the `etregress` command to consider this model.
```s
etregress seconcilio i.calc##i.p_actor d_anio* d_num* d_junta* phase , treat(p_actor = i.time_hr calc d_anio* d_num* d_junta* phase)  cfunction  poutcomes vce(cluster cluster_v)  
```

The Wald test reported in the footer indicates that we can reject the null hypothesis of no correlation
between the EEV-assignment errors and the outcome errors for employee present or abscent. However, the 95% confidence interval for both $\rho_0$ and $\rho_1$ cover $0$. I interpret this as absence of heterogeneity in unobservables. 


#### Maximum likelihood

Analogous to previous sections, we can derive a likelihood function to obtain estimates via ML. 

$$ ln L_j = t_j\left(ln \Phi\left\lbrace\frac{w_j\gamma+(y_j-x_j\beta_1)\rho_1/\sigma_1}{\sqrt{1-\rho_1^2}}\right\rbrace-\frac{1}{2}\left(\frac{y_j-x_j\beta_1}{\sigma_1}\right)^2-ln(\sqrt{2\pi}\sigma_1)\right)$$
$$\quad\quad + (1-t_j)\left(ln \Phi\left\lbrace\frac{-w_j\gamma+(y_j-x_j\beta_0)\rho_0/\sigma_0}{\sqrt{1-\rho_0^2}}\right\rbrace-\frac{1}{2}\left(\frac{y_j-x_j\beta_0}{\sigma_0}\right)^2-ln(\sqrt{2\pi}\sigma_0)\right)$$

```
etregress seconcilio i.calc##i.p_actor d_anio* d_num* d_junta* phase , treat(p_actor = i.time_hr calc d_anio* d_num* d_junta* phase)  poutcomes vce(cluster cluster_v) 
```

In our case, we do not achieve convergence, hence output is not shown.







The joint normality assumption has some limitations,  meaning that they are not robust to violation of this hypothesis. In the same potential outcomes framework, we assume that the binary-EEV follows wither a linear model, or a probit model.


$$t_j = w_j\gamma + u_j$$
$$t_j = 1[w_j\gamma + u_j>0]$$



```s
ivtreatreg seconcilio p_actor calc d_anio* d_num* d_junta* phase, model(probit-ols) iv(time_hr2-time_hr8) hetero(calc) vce(robust) 


ivtreatreg seconcilio p_actor calc d_anio* d_num* d_junta* phase, model(probit-2sls) iv(time_hr2-time_hr8) hetero(calc) vce(robust)


ivtreatreg seconcilio p_actor calc d_anio* d_num* d_junta* phase , model(direct-2sls) iv(time_hr2-time_hr8) hetero(calc) vce(robust)
```  

---
nocite: '@*'
---