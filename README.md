# projecttest
Demo : https://thriving-torte-197d45.netlify.app/


public static Specification<Task> calculateAverageElapsedTimeWithoutDirectJoin() {
    return (Root<Task> root, CriteriaQuery<?> query, CriteriaBuilder cb) -> {
        // Create a subquery or join with a manual condition for RefStatus
        Root<RefStatus> refStatusRoot = query.from(RefStatus.class);

        // Join condition: task.dossierCodeStatus == refStatus.code
        Predicate joinCondition = cb.equal(root.get("dossierCodeStatus"), refStatusRoot.get("code"));

        // Join RefStage from RefStatus
        Join<RefStatus, RefStage> refStageJoin = refStatusRoot.join("refStage", JoinType.INNER);

        // CASE logic for designation
        Expression<String> designation = cb.selectCase()
            .when(cb.equal(root.get("dossierCodeStatus"), "ACCD_RIDN"), "Retour à la charge")
            .otherwise(refStageJoin.get("designation"));

        // Time calculation: COALESCE(exitDate, CURRENT_DATE) - entryDate
        Expression<Long> elapsedTime = cb.diff(
            cb.function("coalesce", Date.class, root.get("exitDate"), cb.literal(Date.from(LocalDate.now().atStartOfDay(ZoneId.systemDefault()).toInstant()))),
            root.get("entryDate")
        );

        Expression<Double> elapsedTimeInHours = cb.quot(
            cb.function("extract_epoch", Long.class, elapsedTime),
            3600
        );

        // Grouping fields
        query.groupBy(designation, refStageJoin.get("code"), refStageJoin.get("id"));

        // Select fields
        query.multiselect(
            designation.alias("designation"),
            refStageJoin.get("code").alias("code"),
            cb.avg(elapsedTimeInHours).alias("averageElapsedTime")
        );

        // Add join condition to the WHERE clause
        query.where(joinCondition);

        // Ordering
        query.orderBy(cb.asc(refStageJoin.get("id")));

        return null; // Predicate not needed for this query
    };
}





***********

import org.springframework.data.jpa.domain.Specification;

import javax.persistence.criteria.*;
import java.time.LocalDate;

public class TaskViewSpecifications {

    public static Specification<TaskView> calculateAverageElapsedTimeWithMinMax(Long specificStatusId) {
        return (Root<TaskView> root, CriteriaQuery<?> query, CriteriaBuilder cb) -> {

            // COALESCE(exitdate, CURRENT_DATE)
            Expression<LocalDate> exitDate = cb.coalesce(root.get("exitDate"), cb.literal(LocalDate.now()));

            // MIN(entryDate) and MAX(exitDate)
            Expression<LocalDate> minEntryDate = cb.least(root.get("entryDate"));
            Expression<LocalDate> maxExitDate = cb.greatest(exitDate);

            // CASE to handle specific status and create 'Fake Stage'
            Expression<String> stageId = cb.selectCase()
                    .when(cb.equal(root.get("status"), specificStatusId), cb.literal("Fake Stage"))
                    .otherwise(root.get("stageId"));

            // Calculate elapsed time: (maxExitDate - minEntryDate) in hours
            Expression<Long> elapsedTimeInSeconds = cb.function(
                    "EXTRACT", Long.class,
                    cb.literal("EPOCH"),
                    cb.diff(maxExitDate, minEntryDate)
            );
            Expression<Double> elapsedTimeInHours = cb.quot(elapsedTimeInSeconds, 3600.0);

            // Group by stage_id
            query.groupBy(stageId);

            // Select stage_id and average elapsed time
            query.multiselect(
                    stageId, // Stage ID or Fake Stage
                    cb.avg(elapsedTimeInHours) // Average elapsed time in hours
            );

            query.orderBy(cb.asc(stageId)); // Order results by stage ID

            return null; // No specific filtering condition
        };
    }
}
