# System Scope

> **Why this document exists:** Scope prevents scope creep and aligns expectations.
> It is equally important to define what the system does NOT do as what it does.
> Review this document at the start of each planning cycle.

---

## In Scope

According to the requirements specification, the FaceAttend EDU project includes the following capabilities:

- User management.
- User registration through CSV files, subject to correction of requirement RF 1.2.
- Sign-in.
- User information consultation.
- Environment management.
- Generation of entry and exit records.
- System configuration.
- Management of IoT device failure alerts, if applicable.
- User role changes.
- Configuration of late arrivals and justifications.
- Generation of attendance and absence histories.
- User change history.
- Upload of justification supporting documents.
- Approval and rejection of justifications.
- Automatic notification of a justification result.
- Report export.
- Analytical dashboard.
- Reports by date range.
- Consultation of late arrivals and absences by parameters.
- School management, including courses and subject curricula.

## Out of Scope

- Teacher payroll or payment processes.
- Complete physical access control for facilities.
- Integration with external enrollment or grading systems.
- Advanced internal messaging functionality.
- Predictive dropout analytics.
- Development of native mobile applications if the project is limited to the web.
- Purchase, installation, or maintenance of physical hardware.

## Assumptions

- Users have access to a compatible web browser.
- The institution has environments or spaces registered in the system.
- Users have roles defined according to their permissions.
- Initial user, group, or environment data can be loaded beforehand.
- The system will support educational management, not serve as the sole legal control mechanism.
- If facial recognition or biometric data is used, the corresponding authorization and data processing will be in place.

## Constraints

- Development is part of a SENA training project.
- It must align with the requirements specification defined by the team.
- It must consider information security and protection criteria.
- It must meet the performance criteria established in the SRS.
- It must be compatible with different browsers or devices, as defined.
- It must respect the time and resources of the academic project.
