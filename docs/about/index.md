# About

## The project

**Lagrange** is the management stack developed by the
[LINKS Foundation](https://linksfoundation.com) Advanced Computing, Photonics and
Electromagnetics research domain to operate Italy's first publicly-accessible
quantum computer — an IQM Spark superconducting system hosted at the
[Politecnico di Torino](https://www.polito.it), in a laboratory run together with
the [Italian National Institute of Metrology (INRIM)](https://www.inrim.it).

It was built in the context of the **QTech Piemonte** strategic initiative, and it
has been open-sourced so that other projects operating quantum hardware — or any
other single, indivisible, accountable instrument — can reuse the platform layer
instead of rebuilding it.

<a href="https://linksfoundation.com"><img src="../assets/logo-links.png" alt="LINKS Foundation" height="60"></a>&nbsp;&nbsp;&nbsp;<a href="https://www.polito.it"><img src="../assets/logo-polito.png" alt="Politecnico di Torino" height="60"></a>&nbsp;&nbsp;&nbsp;<a href="https://www.inrim.it"><img src="../assets/logo-inrim.jpg" alt="INRIM" height="60"></a>

## Status

The stack runs in production. The QC Gateway is publicly released; quantum-api and
quantum-dashboard are being published alongside this documentation.

Nothing in the core is specific to IQM hardware or to the LINKS deployment: vendor
specifics live in a vendor plugin, site specifics in a site plugin and in
configuration, and every behaviour that could reasonably differ between deployments
is an environment variable.

## Known gaps

Stated plainly, because they matter if you deploy this:

- **The backend's machine endpoints are not authenticated.** `/jobReport` is a
  billing write with no credential, and `/jobAuthorizer` verifies the submitting
  user rather than the calling service. Containment is currently the deployment's
  responsibility — see
  [the contract reference](../reference/machine-endpoints.md#authentication-status-and-hardening).
  A service identity for these endpoints is the next significant piece of work, and
  a prerequisite for opening the backend to any client other than its own gateway.
- **The realm-role → platform-role mapping is fixed in code.** `platform-admin` and
  `platform-ro-admin` are hardcoded names in the token exchange. Making the mapping
  configurable is planned; the Pulse-access role is already configurable as a
  precedent.
- **Budget units are inconsistent between two adjacent endpoints.** An
  organization's `initial_budget` is in hours; a project's `budget` is in
  QPU-milliseconds. Both are documented, but it is a wart worth removing.
- **Project tags are platform-wide, not per-organization.** One vocabulary is
  shared by every organization, and only platform admins may extend it — so a
  manager who wants a label of their own has to ask. Scoping tags to an
  organization would need a schema change, which is why the first version is
  deliberately global. There is also no filter by tag on the projects list yet:
  tags are shown as chips and aggregated on the metrics endpoint, but not
  searchable.
- **HPC integration ([Sqed](../hpc/index.md)) is new in 2026.11.0.** It has
  been verified end to end on a containerised SLURM 23.11 cluster against a mock
  quantum computer, not yet on a production HPC system. Mixing Sqed submissions
  with ordinary portal users on one deployment, each with its own accounting, is
  not covered: the tested configuration is HPC-only.

## Contributing

Contributions are welcome, particularly new vendor and site plugins. Each
repository's `CONTRIBUTING.md` has the specifics; [Development](../development/index.md)
is the cross-repository view.

If you deploy this stack somewhere, we would like to know — both because it is
useful to us and because a second deployment is the fastest way to find the places
where "configurable" is still a polite fiction.

## Citation

```bibtex
@misc{viviani2026lagrangeoperatingitalyspubliclyaccessible,
      title={Lagrange: Operating Italy's First Publicly-Accessible Quantum Computer for Research and Education},
      author={Paolo Viviani and Fabrizio Bertone and Giacomo Vitali and Emanuele Dri and Federico Stirano and Giuseppe Caragnano and Francesco Lubrano and Antonino Nespola and Olivier Terzo and Matteo Cocuzza and Bartolomeo Montrucchio and Giovanna Turvani and Gianluca Bertaina and Marco Coisson and Davide Calonico and Fabrizio Pirri and Pietro Asinari},
      year={2026},
      eprint={2604.21695},
      archivePrefix={arXiv},
      primaryClass={quant-ph},
      url={https://arxiv.org/abs/2604.21695},
}
```

The machine itself, and the QPU, are described in *"On-premises superconducting
quantum computer for education and research"*, EPJ Quantum Technology, 2024,
[doi:10.1140/epjqt/s40507-024-00243-z](https://doi.org/10.1140/epjqt/s40507-024-00243-z).
